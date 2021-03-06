<template>
    <div class="resource-layout clearfix">
        <cmdb-hosts-filter class="resource-filter fr"
            :active-tab="['filter']"
            :active-setting="['reset', 'filter-config']"
            :filter-config-key="filter.filterConfigKey"
            @on-refresh="handleRefresh">
            <div class="filter-group" slot="scope">
                <label class="filter-label">{{$t("Hosts['搜索范围']")}}</label>
                <cmdb-form-bool class="filter-field" :disabled="true" :checked="true">
                    <span class="filter-field-label">{{$t("Hosts['未分配主机']")}}</span>
                </cmdb-form-bool>
                <cmdb-form-bool class="filter-field" v-model="filter.assigned">
                    <span class="filter-field-label">{{$t("Hosts['已分配主机']")}}</span>
                </cmdb-form-bool>
            </div>
        </cmdb-hosts-filter>
        <cmdb-hosts-table class="resource-main" ref="resourceTable"
            :columns-config-key="columnsConfigKey"
            :columns-config-properties="columnsConfigProperties"
            :columns-config-disabled-columns="['bk_host_innerip', 'bk_cloud_id', 'bk_biz_name', 'bk_module_name']"
            :edit-auth="$OPERATION.U_RESOURCE_HOST"
            :delete-auth="$OPERATION.D_RESOURCE_HOST"
            :save-auth="$OPERATION.U_RESOURCE_HOST"
            @on-checked="handleChecked"
            @on-set-header="handleSetHeader">
            <div class="resource-options clearfix" slot="options">
                <div class="fl">
                    <span style="display: inline-block;" v-if="isAdminView"
                        v-cursor="{
                            active: !$isAuthorized($OPERATION.C_RESOURCE_HOST),
                            auth: [$OPERATION.C_RESOURCE_HOST]
                        }">
                        <bk-button class="options-button" type="primary" style="margin-left: 0"
                            :disabled="!$isAuthorized($OPERATION.C_RESOURCE_HOST)"
                            @click="importInst.show = true">
                            {{$t('HostResourcePool[\'导入主机\']')}}
                        </bk-button>
                    </span>
                    <cmdb-selector class="options-business-selector"
                        v-if="isAdminView"
                        :placeholder="$t('HostResourcePool[\'分配到业务空闲机池\']')"
                        :disabled="!table.checked.length"
                        :list="authorizedBusiness"
                        :auto-select="false"
                        setting-key="bk_biz_id"
                        display-key="bk_biz_name"
                        v-model="assignBusiness"
                        @on-selected="handleAssignHosts">
                    </cmdb-selector>
                    <span style="display: inline-block;"
                        v-cursor="{
                            active: !$isAuthorized($OPERATION.U_RESOURCE_HOST),
                            auth: [$OPERATION.U_RESOURCE_HOST]
                        }">
                        <bk-button class="options-button" type="default"
                            :disabled="!table.checked.length || !$isAuthorized($OPERATION.U_RESOURCE_HOST)"
                            @click="handleMultipleEdit">
                            {{$t('BusinessTopology[\'修改\']')}}
                        </bk-button>
                    </span>
                    <span style="display: inline-block;" v-if="isAdminView"
                        v-cursor="{
                            active: !$isAuthorized($OPERATION.D_RESOURCE_HOST),
                            auth: [$OPERATION.D_RESOURCE_HOST]
                        }">
                        <bk-button class="options-button options-button-delete" type="default"
                            :disabled="!table.checked.length || !$isAuthorized($OPERATION.D_RESOURCE_HOST)"
                            @click="handleMultipleDelete">
                            {{$t('Common[\'删除\']')}}
                        </bk-button>
                    </span>
                    <bk-button class="options-button" type="default"
                        form="exportForm"
                        :disabled="!table.checked.length"
                        @click="exportField">
                        {{$t('HostResourcePool[\'导出选中\']')}}
                    </bk-button>
                    <cmdb-clipboard-selector class="options-clipboard"
                        :list="clipboardList"
                        :disabled="!table.checked.length"
                        @on-copy="handleCopy">
                    </cmdb-clipboard-selector>
                </div>
                <div class="fr">
                    <bk-button class="options-button options-icon" type="default"
                        v-tooltip="$t('BusinessTopology[\'列表显示属性配置\']')"
                        @click="handleColumnsConfig">
                        <i class="icon-cc-setting"></i>
                    </bk-button>
                    <bk-button class="options-button options-icon" type="default"
                        v-tooltip="$t('Common[\'查看删除历史\']')"
                        @click="routeToHistory">
                        <i class="icon-cc-history"></i>
                    </bk-button>
                </div>
            </div>
        </cmdb-hosts-table>
        <cmdb-slider :is-show.sync="importInst.show" :title="$t('HostResourcePool[\'批量导入\']')">
            <bk-tab :active-name.sync="importInst.active" slot="content">
                <bk-tabpanel name="import" :title="$t('HostResourcePool[\'批量导入\']')">
                    <cmdb-import v-if="importInst.show && importInst.active === 'import'"
                        :template-url="importInst.templateUrl"
                        :import-url="importInst.importUrl"
                        @success="getHostList(true)"
                        @partialSuccess="getHostList(true)">
                        <span slot="download-desc" style="display: inline-block;vertical-align: top;">
                            {{$t('HostResourcePool["说明：内网IP为必填列"]')}}
                        </span>
                    </cmdb-import>
                </bk-tabpanel>
                <bk-tabpanel name="agent" :title="$t('HostResourcePool[\'自动导入\']')">
                    <div class="automatic-import">
                        <p>{{$t("HostResourcePool['agent安装说明']")}}</p>
                        <div class="back-contain">
                            <i class="icon-cc-skip"></i>
                            <a href="javascript:void(0)" @click="openAgentApp">{{$t("HostResourcePool['点此进入节点管理']")}}</a>
                        </div>
                    </div>
                </bk-tabpanel>
            </bk-tab>
        </cmdb-slider>
    </div>
</template>

<script>
    import { mapGetters, mapActions } from 'vuex'
    import cmdbHostsFilter from '@/components/hosts/filter'
    import cmdbHostsTable from '@/components/hosts/table'
    import cmdbImport from '@/components/import/import'
    export default {
        components: {
            cmdbHostsFilter,
            cmdbHostsTable,
            cmdbImport
        },
        data () {
            return {
                properties: {
                    biz: [],
                    host: [],
                    set: [],
                    module: []
                },
                table: {
                    checked: [],
                    header: [],
                    columnsConfigKey: 'resource_table_columns',
                    exportUrl: `${window.API_HOST}hosts/export`
                },
                filter: {
                    filterConfigKey: 'resource_filter_fields',
                    business: -1,
                    assigned: false,
                    params: null,
                    paramsResolver: null
                },
                assignBusiness: '',
                importInst: {
                    show: false,
                    active: 'import',
                    templateUrl: `${window.API_HOST}importtemplate/host`,
                    importUrl: `${window.API_HOST}hosts/import`
                }
            }
        },
        computed: {
            ...mapGetters(['userName', 'isAdminView']),
            ...mapGetters('userCustom', ['usercustom']),
            ...mapGetters('objectBiz', ['authorizedBusiness', 'bizId']),
            columnsConfigKey () {
                return `${this.userName}_$resource_${this.isAdminView ? 'adminView' : this.bizId}_table_columns`
            },
            customColumns () {
                return this.usercustom[this.columnsConfigKey]
            },
            clipboardList () {
                return this.table.header.filter(header => header.type !== 'checkbox')
            },
            columnsConfigProperties () {
                const setProperties = this.properties.set.filter(property => ['bk_set_name'].includes(property['bk_property_id']))
                const moduleProperties = this.properties.module.filter(property => ['bk_module_name'].includes(property['bk_property_id']))
                const businessProperties = this.properties.biz.filter(property => ['bk_biz_name'].includes(property['bk_property_id']))
                const hostProperties = this.properties.host
                return [...setProperties, ...moduleProperties, ...businessProperties, ...hostProperties]
            }
        },
        watch: {
            'importInst.show' (show) {
                if (!show) {
                    this.importInst.active = 'import'
                }
            }
        },
        async created () {
            this.$store.commit('setHeaderTitle', this.$t('Nav["主机"]'))
            try {
                this.$store.dispatch('userCustom/setRencentlyData', { id: 'resource' })
                this.setQueryParams()
                await Promise.all([
                    this.getParams(),
                    this.getProperties()
                ])
                this.getHostList()
            } catch (e) {
                console.log(e)
            }
        },
        methods: {
            ...mapActions('hostBatch', ['exportHost']),
            ...mapActions('hostSearch', ['searchHost']),
            ...mapActions('hostDelete', ['deleteHost']),
            ...mapActions('hostRelation', ['transferResourcehostToIdleModule']),
            ...mapActions('objectModelProperty', ['batchSearchObjectAttribute']),
            setQueryParams () {
                const query = this.$route.query
                if (query.hasOwnProperty('assigned')) {
                    this.filter.assigned = ['true', 'false'].includes(query.assigned) ? query.assigned === 'true' : !!query.assigned
                }
            },
            getParams () {
                return new Promise((resolve, reject) => {
                    this.filter.paramsResolver = () => {
                        this.filter.paramsResolver = null
                        resolve()
                    }
                })
            },
            getProperties () {
                return this.batchSearchObjectAttribute({
                    params: this.$injectMetadata({
                        bk_obj_id: { '$in': Object.keys(this.properties) },
                        bk_supplier_account: this.supplierAccount
                    }, { inject: false }),
                    config: {
                        requestId: `post_batchSearchObjectAttribute_${Object.keys(this.properties).join('_')}`,
                        requestGroup: Object.keys(this.properties).map(id => `post_searchObjectAttribute_${id}`)
                    }
                }).then(result => {
                    Object.keys(this.properties).forEach(objId => {
                        this.properties[objId] = result[objId]
                    })
                    return result
                })
            },
            handleRefresh (params) {
                this.filter.params = params
                if (this.filter.paramsResolver) {
                    this.filter.paramsResolver()
                } else {
                    this.getHostList(true)
                }
            },
            getHostList (resetPage = false) {
                this.$refs.resourceTable.search(this.filter.business, this.getScopedParams(), resetPage)
            },
            getScopedParams () {
                const params = this.$tools.clone(this.filter.params)
                if (!this.filter.assigned) {
                    const businessParams = params.condition.find(condition => condition['bk_obj_id'] === 'biz')
                    businessParams.condition.push({
                        field: 'default',
                        operator: '$eq',
                        value: 1
                    })
                }
                return params
            },
            routeToHistory () {
                this.$router.push({
                    name: 'history',
                    params: {
                        objId: 'host'
                    }
                })
            },
            handleAssignHosts (businessId, business) {
                if (!businessId) return
                if (this.hasSelectAssignedHost()) {
                    this.$error(this.$t('Hosts["请勿选择已分配主机"]'))
                    this.assignBusiness = ''
                } else {
                    this.$bkInfo({
                        title: this.$t("HostResourcePool['请确认是否转移']"),
                        content: this.getConfirmContent(business),
                        confirmFn: () => {
                            this.assignHosts(business)
                        },
                        cancelFn: () => {
                            this.assignBusiness = ''
                        }
                    })
                }
            },
            hasSelectAssignedHost () {
                const allList = this.$refs.resourceTable.table.allList
                const list = allList.filter(item => this.table.checked.includes(item['host']['bk_host_id']))
                const existAssigned = list.some(item => item['biz'].some(biz => biz.default !== 1))
                return existAssigned
            },
            assignHosts (business) {
                this.transferResourcehostToIdleModule({
                    params: {
                        'bk_biz_id': business['bk_biz_id'],
                        'bk_host_id': this.table.checked
                    }
                }).then(() => {
                    this.$success(this.$t("HostResourcePool['分配成功']"))
                    this.assignBusiness = ''
                    this.$refs.resourceTable.table.checked = []
                    this.$refs.resourceTable.handlePageChange(1)
                }).catch(e => {
                    this.assignBusiness = ''
                })
            },
            getConfirmContent (business) {
                const render = this.$createElement
                let content
                if (this.$i18n.locale === 'en') {
                    content = render('p', [
                        render('span', 'Selected '),
                        render('span', {
                            style: { color: '#3c96ff' }
                        }, this.table.checked.length),
                        render('span', ' Hosts Transfer to Idle machine under '),
                        render('span', {
                            style: { color: '#3c96ff' }
                        }, business['bk_biz_name'])
                    ])
                } else {
                    content = render('p', [
                        render('span', '选中的 '),
                        render('span', {
                            style: { color: '#3c96ff' }
                        }, this.table.checked.length),
                        render('span', ' 个主机转移到 '),
                        render('span', {
                            style: { color: '#3c96ff' }
                        }, business['bk_biz_name']),
                        render('span', ' 下的空闲机模块')
                    ])
                }
                return content
            },
            handleChecked (checked) {
                this.table.checked = checked
            },
            handleSetHeader (header) {
                this.table.header = header
            },
            handleColumnsConfig () {
                this.$refs.resourceTable.columnsConfig.show = true
            },
            handleCopy (target) {
                this.$refs.resourceTable.handleCopy(target)
            },
            handleMultipleEdit () {
                if (this.hasSelectAssignedHost()) {
                    this.$error(this.$t('Hosts["请勿选择已分配主机"]'))
                    return false
                }
                this.$refs.resourceTable.handleMultipleEdit()
            },
            handleMultipleDelete () {
                if (this.hasSelectAssignedHost()) {
                    this.$error(this.$t('Hosts["请勿选择已分配主机"]'))
                    return false
                }
                this.$bkInfo({
                    title: `${this.$t("HostResourcePool['确定删除选中的主机']")}？`,
                    confirmFn: () => {
                        this.deleteHost({
                            params: {
                                data: {
                                    'bk_host_id': this.table.checked.join(','),
                                    'bk_supplier_account': this.supplierAccount
                                }
                            }
                        }).then(() => {
                            this.$success(this.$t("HostResourcePool['成功删除选中的主机']"))
                            this.$refs.resourceTable.table.checked = []
                            this.$refs.resourceTable.handlePageChange(1)
                        })
                    }
                })
            },
            openAgentApp () {
                const agent = window.Site.agent
                if (agent) {
                    const topWindow = window.top
                    const isPaasConsole = topWindow !== window
                    if (isPaasConsole) {
                        topWindow.postMessage(JSON.stringify({
                            action: 'open_other_app',
                            app_code: 'bk_nodeman'
                        }), '*')
                    } else {
                        window.open(agent)
                    }
                } else {
                    this.$warn(this.$t("HostResourcePool['未配置Agent安装APP地址']"))
                }
            },
            exportExcel (response) {
                const contentDisposition = response.headers['content-disposition']
                const fileName = contentDisposition.substring(contentDisposition.indexOf('filename') + 9)
                const url = window.URL.createObjectURL(new Blob([response.data], { type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet' }))
                const link = document.createElement('a')
                link.style.display = 'none'
                link.href = url
                link.setAttribute('download', fileName)
                document.body.appendChild(link)
                link.click()
                document.body.removeChild(link)
            },
            async exportField () {
                const formData = new FormData()
                formData.append('bk_host_id', this.table.checked)
                if (this.customColumns) {
                    formData.append('export_custom_fields', this.customColumns)
                }
                formData.append('bk_biz_id', '-1')
                const res = await this.exportHost({
                    params: formData,
                    config: {
                        globalError: false,
                        originalResponse: true,
                        responseType: 'blob'
                    }
                })
                this.exportExcel(res)
            }
        }
    }
</script>

<style lang="scss" scoped>
    .resource-layout{
        height: 100%;
        padding: 0;
        overflow: hidden;
        .resource-main{
            height: 100%;
            padding: 20px;
            overflow: hidden;
        }
        .resource-filter{
            height: 100%;
            .filter-field{
                margin: 10px 15px 0 0;
            }
            .filter-field-label{
                display: inline-block;
                vertical-align: middle;
                line-height: 1;
                font-size: 14px;
            }
        }
    }
    .resource-options{
        font-size: 0;
        .options-button{
            position: relative;
            display: inline-block;
            vertical-align: middle;
            font-size: 14px;
            margin: 0 5px;
            padding: 0 10px;
            &:hover{
                z-index: 1;
            }
            &-delete:hover{
                border-color: #ef4c4c;
                color: #ef4c4c;
            }
            &-history{
                width: 36px;
                padding: 0;
                margin: 0 0 0 10px;
                border-radius: 2px;
            }
            &.options-icon {
                border-radius: 0;
                margin: 0 -1px 0 0;
            }
        }
        .options-clipboard {
            margin: 0 5px;
        }
        .options-table-selector,
        .options-business-selector{
            margin: 0 5px 0 0;
        }
        .options-business-selector{
            width: 180px;
        }
    }
    .resource-table{
        margin-top: 20px;
    }
    .automatic-import{
        padding:40px 30px 0 30px;
        .back-contain{
            cursor:pointer;
            color: #3c96ff;
            img{
                margin-right: 5px;
            }
            a{
                color:#3c96ff;
            }
        }
    }
</style>
