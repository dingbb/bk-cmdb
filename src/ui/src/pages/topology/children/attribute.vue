<template>
    <div class="topo-attribute" v-bkloading="{isLoading: isLoading || attributeLoading}">
        <ul class="attribute-list">
            <template v-for="(property, index) in attribute[bkObjId]">
                <li class="attribute-item clearfix" v-if="!property['bk_isapi']">
                    <div v-show="displayType === 'form'">
                        <label class="attribute-item-label fl" :class="{'required': property['isrequired']}">
                            {{property['bk_property_name']}}
                        </label>
                        <div class="attribute-item-field fl" :style="{zIndex: attribute[bkObjId].length - index}">
                            <input v-if="property['bk_property_type'] === 'int'" 
                                type="number" class="bk-form-input"
                                :disabled="!property['editable']"
                                v-model.number="localValues[property['bk_property_id']]">
                            <v-member-selector v-else-if="property['bk_property_type'] === 'objuser'"
                                :selected.sync="localValues[property['bk_property_id']]"
                                :disabled="!property['editable']" 
                                :multiple="true">
                            </v-member-selector>
                            <bk-datepicker style="width: 100%;" v-else-if="property['bk_property_type'] === 'date'"
                                :timer="false"
                                :disabled="!property['editable']"
                                :init-date="localValues[property['bk_property_id']]"
                                @date-selected="setDate(...arguments, property['bk_property_id'])">
                            </bk-datepicker>
                            <bk-datepicker style="width: 100%;" v-else-if="property['bk_property_type'] === 'time'"
                                :timer="true"
                                :disabled="!property['editable']"
                                :init-date="localValues[property['bk_property_id']]"
                                @date-selected="setDate(...arguments, property['bk_property_id'])">
                            </bk-datepicker>
                            <v-association v-else-if="property['bk_property_type'] === 'singleasst' || property['bk_property_type'] === 'multiasst'"
                                :asstObjId="property['bk_asst_obj_id']"
                                :multiple="property['bk_property_type'] === 'multiasst'"
                                :disabled="!property['editable']"
                                :selected.sync="localValues[property['bk_property_id']]">
                            </v-association>
                            <v-enumeration v-else-if="property['bk_property_type'] === 'enum'"
                                :disabled="!property['editable']"
                                :selected.sync="localValues[property['bk_property_id']]"
                                :options="property['option']">
                            </v-enumeration>
                            <v-timezone v-else-if="property['bk_property_type'] === 'timezone'"
                                :selected.sync="localValues[property['bk_property_id']]"
                                :disabled="!property['editable']">
                            </v-timezone>
                            <span class="bk-form-checkbox" v-else-if="property['bk_property_type'] === 'bool'">
                                <input type="checkbox" v-model="localValues[property['bk_property_id']]" :disabled="!property['editable']">
                            </span>
                            <input v-else
                                :disabled="!property['editable']"
                                type="text" class="bk-form-input"
                                v-model.trim="localValues[property['bk_property_id']]">
                            <!-- <v-validate class="attribute-validate-result" v-if="checkIsNeedValidate(property)" -->
                            <v-validate class="attribute-validate-result"
                                v-validate="getValidateRules(property)"
                                :name="property['bk_property_name']" 
                                :value="localValues[property['bk_property_id']]">
                            </v-validate>
                        </div>
                    </div>
                    <div v-show="displayType === 'list'">
                        <label class="attribute-item-label list fl">
                            {{property['bk_property_name']}}
                        </label>
                        <div class="attribute-item-field list fl">
                            <template v-if="property['bk_property_type'] === 'singleasst' || property['bk_property_type'] === 'multiasst'">
                                {{getAsstLabel(formValues[property['bk_property_id']])}}
                            </template>
                            <template v-else>
                                {{localValues[property['bk_property_id']]}}
                            </template>
                        </div>
                    </div>
                </li>
            </template>
        </ul>
        <div class="attribute-btn">
            <bk-button type="primary" class="bk-button main-btn" @click="doSubmit" :disabled="errors.any()" v-if="displayType === 'form'">保存</bk-button>
            <bk-button type="primary" class="bk-button main-btn" @click="toggleDisplayType('form')" v-if="displayType === 'list'">编辑</bk-button>
            <bk-button type="default" class="bk-button vice-btn cancel-btn" @click="toggleDisplayType('list')" v-if="type === 'update' && displayType === 'form'">取消</bk-button>
            <bk-button type="default" class="bk-button vice-btn cancel-btn" @click="cancelCreate" v-if="type === 'create'">取消</bk-button>
            <button class="del-btn" @click="doDelete" v-if="type === 'update' && displayType === 'form'">删除</button>
        </div>
    </div>
</template>
<script type="text/javascript">
    import vMemberSelector from '@/components/common/selector/member'
    import vEnumeration from '@/components/common/selector/enumeration'
    import vAssociation from '@/components/common/selector/association'
    import vValidate from '@/components/common/validator/validate'
    import { mapGetters } from 'vuex'
    const vTimezone = () => import('@/components/timezone/timezone')
    export default {
        props: {
            bkObjId: String,
            bkBizId: Number,
            activeNode: Object,
            activeParentNode: Object,
            type: {
                type: String,
                default: 'create'
            },
            active: {
                type: Boolean,
                default: false
            },
            formValues: {
                type: Object,
                default () {
                    return {}
                }
            },
            isLoading: {
                type: Boolean,
                default: false
            }
        },
        data () {
            return {
                attribute: {},
                localValues: {},
                displayType: 'list',
                attributeLoading: false
            }
        },
        computed: {
            ...mapGetters(['bkSupplierAccount']),
            shouldInitFormValues () {
                let isAttributeLoaded = this.attribute.hasOwnProperty(this.bkObjId)
                let hasInitValues = !!Object.keys(this.formValues).length
                return isAttributeLoaded && hasInitValues
            }
        },
        watch: {
            bkObjId (bkObjId) {
                if (!this.attribute.hasOwnProperty(bkObjId)) {
                    this.getAttribute()
                }
            },
            shouldInitFormValues (shouldInit) {
                if (shouldInit) {
                    this.initLocalValues()
                }
            },
            type (type) {
                if (type === 'create') {
                    this.initLocalValues()
                    this.toggleDisplayType('form')
                    this.$validator.validateAll().then(() => {
                        this.errors.clear()
                    })
                }
            },
            activeNode (activeNode) {
                this.displayType = 'list'
            },
            formValues () {
                this.initLocalValues()
            },
            active (active) {
                if (!active) {
                    this.toggleDisplayType('list')
                    setTimeout(() => {
                        this.localValues = {}
                        this.$validator.validateAll().then(() => {
                            this.errors.clear()
                        })
                    }, 100)
                } else {
                    this.initLocalValues()
                    if (this.type === 'create') {
                        this.toggleDisplayType('form')
                    }
                }
            },
            displayType (displayType) {
                if (displayType === 'list') {
                    this.initLocalValues()
                }
            }
        },
        methods: {
            toggleDisplayType (displayType) {
                this.displayType = displayType
            },
            initLocalValues () {
                let specialInitType = {
                    'int': null,
                    'enum': null,
                    'date': null,
                    'time': null,
                    'bool': false,
                    'timezone': null
                }
                this.localValues = {}
                if (this.shouldInitFormValues) {
                    this.attribute[this.bkObjId].map(property => {
                        let {
                            bk_property_id: bkPropertyId,
                            bk_asst_obj_id: bkAsstObjId,
                            bk_property_type: bkPropertyType
                        } = property
                        if (this.formValues.hasOwnProperty(bkPropertyId)) {
                            if (specialInitType.hasOwnProperty(bkPropertyType) && this.formValues[bkPropertyId] === null) {
                                this.$set(this.localValues, bkPropertyId, '')
                            } else {
                                this.$set(this.localValues, bkPropertyId, this.formValues[bkPropertyId])
                            }
                        }
                    })
                }
            },
            getAttribute () {
                this.attributeLoading = true
                this.$axios.post('object/attr/search', {
                    'bk_obj_id': this.bkObjId,
                    'bk_supplier_account': this.bkSupplierAccount
                }).then(res => {
                    if (res.result) {
                        this.$set(this.attribute, this.bkObjId, res.data)
                    } else {
                        this.$alertMsg(res['bk_error_msg'])
                    }
                    this.attributeLoading = false
                }).catch(() => {
                    this.attributeLoading = false
                })
            },
            getAsstLabel (value) {
                if (Array.isArray(value)) {
                    return value.map(({bk_inst_name: bkInstName}) => bkInstName).join(',')
                }
                return value
            },
            setDate (date, bkPropertyId) {
                if (this.localValues.hasOwnProperty(bkPropertyId)) {
                    this.localValues[bkPropertyId] = date
                } else {
                    this.$set(this.localValues, bkPropertyId, date)
                }
            },
            checkIsNeedValidate (property) {
                let isNeedValidate = property['isrequired']
                let validateOptionType = ['int', 'singlechar', 'longchar']
                let bkPropertyType = property['bk_property_type']
                if (property['option'] && validateOptionType.indexOf(bkPropertyType) !== -1) {
                    isNeedValidate = true
                }
                return isNeedValidate
            },
            getValidateRules (property) {
                let rules = {}
                let {
                    bk_property_type: bkPropertyType,
                    isrequired,
                    option
                } = property
                if (isrequired) {
                    rules['required'] = true
                }
                if (property.hasOwnProperty('option')) {
                    if (bkPropertyType === 'int') {
                        option = JSON.parse(option)
                        if (option.hasOwnProperty('min')) {
                            rules['min_value'] = option.min
                        }
                        if (option.hasOwnProperty('max')) {
                            rules['max_value'] = option.max
                        }
                    } else if (bkPropertyType === 'singlechar' || bkPropertyType === 'longchar') {
                        rules['regex'] = option
                    }
                }
                if (bkPropertyType === 'singlechar' || bkPropertyType === 'longchar') {
                    rules['char'] = true
                }
                return rules
            },
            getFormData () {
                let formData = Object.assign({}, this.localValues)
                this.attribute[this.bkObjId].map(property => {
                    let {
                        bk_isapi: bkIsapi,
                        bk_property_type: bkPropertyType,
                        bk_property_id: bkPropertyId
                    } = property
                    let specialInitValue = {
                        'int': null,
                        'enum': null,
                        'date': null,
                        'time': null,
                        'bool': false,
                        'timezone': null
                    }
                    if (bkIsapi) {
                        if (bkPropertyId === 'bk_biz_id') {
                            formData[bkPropertyId] = this.bkBizId
                        } else if (bkPropertyId === 'bk_set_id') {
                            if (this.bkObjId === 'set') {
                                formData[bkPropertyId] = this.activeNode['bk_inst_id']
                            } else if (this.bkObjId === 'module') {
                                formData[bkPropertyId] = this.activeParentNode['bk_inst_id']
                            }
                        } else {
                            formData[bkPropertyId] = null
                        }
                    } else {
                        if (specialInitValue.hasOwnProperty(bkPropertyType)) {
                            if (!formData.hasOwnProperty(bkPropertyId) || formData[bkPropertyId] === '') {
                                formData[bkPropertyId] = specialInitValue[bkPropertyType]
                            }
                        } else if (!formData.hasOwnProperty(bkPropertyId)) {
                            formData[bkPropertyId] = ''
                        }
                    }
                })
                return formData
            },
            doSubmit () {
                this.$validator.validateAll().then(isValid => {
                    if (isValid) {
                        this.$emit('submit', this.getFormData(), this.formValues)
                    }
                })
            },
            doDelete () {
                this.$emit('delete', this.formValues)
            },
            cancelCreate () {
                this.$emit('cancel')
            }
        },
        components: {
            vMemberSelector,
            vValidate,
            vTimezone,
            vEnumeration,
            vAssociation
        }
    }
</script>

<style lang="scss" scoped>
    .attribute-list{
        padding: 20px 0 30px;
        .attribute-item{
            margin: 20px 0 0 0;
            font-size: 14px;
            color: #737987;
            .attribute-item-label{
                width: 145px;
                line-height: 36px;
                display: inline-block;
                text-align: right;
                padding-right: 27px;
                vertical-align: top;
                position: relative;
                &.required:after{
                    content: "*";
                    color: #ff5656;
                    position: absolute;
                    right: 18px;
                    top: 3px;
                }
                &.list:after{
                    content: ":";
                    position: absolute;
                    right: 18px;
                    top: 0;
                }
            }
            .attribute-item-field{
                width: 460px;
                height: 36px;
                line-height: 36px;
                position: relative;
                &.list{
                    padding: 0 11px;
                }
                .attribute-validate-result{
                    position: absolute;
                    top: 100%;
                    line-height: 16px;
                }
            }
        }
    }
    .attribute-btn{
        padding: 0 0 0 145px;
        .bk-button,
        .del-btn{
            vertical-align: middle;
            // width: 124px;
            font-size: 14px;
            height: 36px;
            line-height: 34px;
            margin: 0 15px 0 0;
        }
        .cancel-btn {
            // width: 124px;
        }
    }
    .bk-form-input{
        vertical-align: top;
        line-height: 36px;
    }
</style>

<style lang="scss">
    .topo-attribute{
        .attribute-item-field{
            .date-dropdown-panel{
                width: 260px;
            }
        }
    }
</style>