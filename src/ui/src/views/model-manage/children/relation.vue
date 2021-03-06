<template>
    <div class="model-relation-wrapper">
        <span v-cursor="{
            active: !$isAuthorized($OPERATION.U_MODEL),
            auth: [$OPERATION.U_MODEL]
        }">
            <bk-button class="create-btn" type="primary"
                :disabled="isReadOnly || !updateAuth"
                @click="createRelation">
                {{$t('ModelManagement["新建关联"]')}}
            </bk-button>
        </span>
        <cmdb-table
            class="relation-table"
            :loading="$loading()"
            :sortable="false"
            :header="table.header"
            :list="table.list"
            :pagination.sync="table.pagination"
            :wrapper-minus-height="220"
            @handleSortChange="handleSortChange">
            <template slot="bk_obj_asst_id" slot-scope="{ item }">
                <span
                    v-if="item.ispre"
                    :class="['relation-pre', $i18n.locale]">
                    {{$t('ModelManagement["内置"]')}}
                </span>
                <span class="relation-id">{{item['bk_obj_asst_id']}}</span>
            </template>
            <template slot="bk_asst_name" slot-scope="{ item }">
                {{getRelationName(item['bk_asst_id'])}}
            </template>
            <template slot="mapping" slot-scope="{ item }">
                {{mappingMap[item.mapping]}}
            </template>
            <template slot="bk_obj_name" slot-scope="{ item }">
                {{getModelName(item['bk_obj_id'])}}
            </template>
            <template slot="bk_asst_obj_name" slot-scope="{ item }">
                {{getModelName(item['bk_asst_obj_id'])}}
            </template>
            <template slot="operation" slot-scope="{ item }">
                <button class="text-primary mr10"
                    :disabled="!isEditable(item)"
                    @click.stop="editRelation(item)">
                    {{$t('Common["编辑"]')}}
                </button>
                <button class="text-primary"
                    :disabled="!isEditable(item)"
                    @click.stop="deleteRelation(item)">
                    {{$t('Common["删除"]')}}
                </button>
            </template>
        </cmdb-table>
        <cmdb-slider
            :width="450"
            :title="slider.title"
            :is-show.sync="slider.isShow">
            <the-relation-detail
                class="slider-content"
                slot="content"
                :is-read-only="isReadOnly"
                :is-edit="slider.isEdit"
                :relation="slider.relation"
                :relation-list="relationList"
                @save="saveRelation"
                @cancel="slider.isShow = false">
            </the-relation-detail>
        </cmdb-slider>
    </div>
</template>

<script>
    import theRelationDetail from './relation-detail'
    import { mapGetters, mapActions } from 'vuex'
    export default {
        components: {
            theRelationDetail
        },
        data () {
            return {
                slider: {
                    isShow: false,
                    isEdit: false,
                    title: this.$t('ModelManagement["新建关联"]'),
                    relation: {}
                },
                relationList: [],
                table: {
                    header: [{
                        id: 'bk_obj_asst_id',
                        name: this.$t('ModelManagement["唯一标识"]')
                    }, {
                        id: 'bk_asst_name',
                        name: this.$t('ModelManagement["关联类型"]')
                    }, {
                        id: 'mapping',
                        name: this.$t('ModelManagement["源-目标约束"]')
                    }, {
                        id: 'bk_obj_name',
                        name: this.$t('ModelManagement["源模型"]')
                    }, {
                        id: 'bk_asst_obj_name',
                        name: this.$t('ModelManagement["目标模型"]')
                    }, {
                        id: 'operation',
                        name: this.$t('Common["操作"]')
                    }],
                    list: [],
                    defaultSort: '-op_time',
                    sort: '-op_time'
                },
                mappingMap: {
                    '1:1': '1-1',
                    '1:n': '1-N',
                    'n:n': 'N-N'
                }
            }
        },
        computed: {
            ...mapGetters(['isAdminView', 'isBusinessSelected']),
            ...mapGetters('objectModel', [
                'activeModel',
                'isInjectable'
            ]),
            ...mapGetters('objectModelClassify', ['models']),
            isReadOnly () {
                if (this.activeModel) {
                    return this.activeModel['bk_ispaused']
                }
                return false
            },
            updateAuth () {
                const cantEdit = ['process', 'plat']
                if (cantEdit.includes(this.$route.params.modelId)) {
                    return false
                }
                const editable = this.isAdminView || (this.isBusinessSelected && this.isInjectable)
                return editable && this.$isAuthorized(this.$OPERATION.U_MODEL)
            }
        },
        created () {
            if (!this.updateAuth) {
                this.table.header.pop()
            }
            this.searchRelationList()
            this.initRelationList()
        },
        methods: {
            ...mapActions('objectAssociation', [
                'searchObjectAssociation',
                'deleteObjectAssociation',
                'searchAssociationType'
            ]),
            isEditable (item) {
                if (item.ispre || item['bk_asst_id'] === 'bk_mainline' || this.isReadOnly) {
                    return false
                }
                if (!this.isAdminView) {
                    return !!this.$tools.getMetadataBiz(item)
                }
                return true
            },
            getRelationName (id) {
                const relation = this.relationList.find(item => item.id === id)
                if (relation) {
                    return relation.name
                }
            },
            async initRelationList () {
                const data = await this.searchAssociationType({
                    params: {},
                    config: {
                        requestId: 'post_searchAssociationType',
                        fromCache: true
                    }
                })
                this.relationList = data.info.map(({ bk_asst_id: asstId, bk_asst_name: asstName }) => {
                    if (asstName.length) {
                        return {
                            id: asstId,
                            name: `${asstId}(${asstName})`
                        }
                    }
                    return {
                        id: asstId,
                        name: asstId
                    }
                })
            },
            getModelName (objId) {
                const model = this.models.find(model => model['bk_obj_id'] === objId)
                if (model) {
                    return model['bk_obj_name']
                }
                return ''
            },
            createRelation () {
                this.slider.isEdit = false
                this.slider.isReadOnly = false
                this.slider.relation = {}
                this.slider.title = this.$t('ModelManagement["新建关联"]')
                this.slider.isShow = true
            },
            editRelation (item) {
                this.slider.isEdit = true
                this.slider.isReadOnly = false
                this.slider.relation = item
                this.slider.title = this.$t('ModelManagement["编辑关联"]')
                this.slider.isShow = true
            },
            deleteRelation (relation) {
                this.$bkInfo({
                    title: this.$t('ModelManagement["确定删除关联关系?"]'),
                    confirmFn: async () => {
                        await this.deleteObjectAssociation({
                            id: relation.id,
                            config: {
                                data: this.$injectMetadata({}, { inject: this.isInjectable }),
                                requestId: 'deleteObjectAssociation'
                            }
                        }).then(() => {
                            this.$http.cancel(`post_searchObjectAssociation_${this.activeModel['bk_obj_id']}`)
                        })
                        this.searchRelationList()
                    }
                })
            },
            async searchRelationList () {
                const [source, dest] = await Promise.all([this.searchAsSource(), this.searchAsDest()])
                this.table.list = [...source, ...dest]
            },
            searchAsSource () {
                return this.searchObjectAssociation({
                    params: this.$injectMetadata({
                        condition: {
                            'bk_obj_id': this.activeModel['bk_obj_id']
                        }
                    }, {
                        inject: this.isInjectable
                    })
                })
            },
            searchAsDest () {
                return this.searchObjectAssociation({
                    params: this.$injectMetadata({
                        condition: {
                            'bk_asst_obj_id': this.activeModel['bk_obj_id']
                        }
                    }, {
                        inject: this.isInjectable
                    })
                })
            },
            saveRelation () {
                this.slider.isShow = false
                this.searchRelationList()
            },
            handleSortChange (sort) {
                this.table.sort = sort
            }
        }
    }
</script>

<style lang="scss" scoped>
    .create-btn {
        margin: 10px 0;
    }
    .relation-pre {
        display: inline-block;
        margin-right: -26px;
        padding: 0 6px;
        vertical-align: middle;
        line-height: 32px;
        border-radius: 4px;
        background-color: #a4aab3;
        color: #fff;
        font-size: 20px;
        transform: scale(0.5);
        transform-origin: left center;
        opacity: 0.4;
        &.en {
            margin-right: -40px;
        }
    }
    .relation-id {
        display: inline-block;
        vertical-align: middle;
    }
    .text-primary {
        cursor: pointer;
    }
</style>
