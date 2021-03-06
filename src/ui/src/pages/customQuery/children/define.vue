/*
 * Tencent is pleased to support the open source community by making 蓝鲸 available.
 * Copyright (C) 2017-2018 THL A29 Limited, a Tencent company. All rights reserved.
 * Licensed under the MIT License (the "License"); you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at http://opensource.org/licenses/MIT
 * Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and limitations under the License.
 */

<template>
    <div class="userapi-wrapper" id="userapiWrapper">
        <div class="userapi-group">
            <div class="userapi-input clearfix">
                <label class="userapi-input-name fl">{{$t("CustomQuery['查询名称']")}}</label>
                <input type="text" class="bk-form-input userapi-input-text fl" maxlength="15" 
                    v-model.trim="name">
                <v-validate class="validate-message" v-validate="'required|max:15'" :name="$t('CustomQuery[\'查询名称\']')" :value="name"></v-validate>
            </div>
        </div>
        <div class="userapi-group">
            <div class="userapi-input clearfix">
                <label class="userapi-input-name fl">{{$t("CustomQuery['查询内容']")}}</label>
                <bk-select class="fl userapi-content-selector"
                    :selected.sync="attribute.selected"
                    :filterable="true"
                    :multiple="true">
                    <bk-select-option v-for="(property, index) in attribute.list"
                        :key="index"
                        :disabled="property.disabled"
                        :value="property['bk_property_id']"
                        :label="property['bk_property_name']">
                    </bk-select-option>
                </bk-select>
            </div>
        </div>
        <div class="userapi-group">
            <ul class="userapi-list">
                <li class="userapi-item clearfix" v-for="(property, index) in userProperties" :style="{zIndex: userProperties.length - index}">
                    <label class="userapi-name fl">{{property.bkPropertyName}}</label>
                    <span v-if="property.bkPropertyType === 'time'">
                        <bk-daterangepicker class="userapi-date fl"
                            :range-separator="'-'"
                            :quickSelect="false"
                            :timer="true"
                            :start-date="property.value.split(' - ')[0]"
                            :end-date="property.value.split(' - ')[1]"
                            :init-date="property.value"
                            @change="setUserPropertyTime(...arguments, index)">
                        </bk-daterangepicker>
                    </span>
                    <span v-else-if="property.bkPropertyType === 'date'">
                        <bk-daterangepicker class="userapi-date fl"
                            :range-separator="'-'"
                            :quickSelect="false"
                            :start-date="property.value.split(' - ')[0]"
                            :end-date="property.value.split(' - ')[1]"
                            :init-date="property.value"
                            @change="setUserPropertyTime(...arguments, index)">
                        </bk-daterangepicker>
                    </span>
                    <span v-else-if="property.bkPropertyType === 'enum'">
                        <bk-select :selected.sync="property.value" class="userapi-enum fl">
                            <bk-select-option v-for="option in getEnumOptions(property)"
                                :key="option.id"
                                :value="option.id"
                                :label="option.name">
                            </bk-select-option>
                        </bk-select>
                    </span>
                    <span v-else>
                        <v-operator 
                            :property="property"
                            :type="property.bkPropertyType"
                            :selected.sync="property.operator">
                        </v-operator>
                        <input type="text" maxlength="11" class="userapi-text fl"
                            v-if="property.bkPropertyType === 'int'" 
                            v-model.number="property.value">
                        <input v-else type="text" class="userapi-text fl"
                            v-model.trim="property.value">
                    </span>
                    <i class="icon icon-cc-del userapi-delete fl" @click="deleteUserProperty(property, index)"></i>
                    <v-validate class="validate-message" v-validate="'required'"
                        :name="property.bkPropertyName" 
                        :value="property.value">
                    </v-validate>
                </li>
            </ul>
            <div class="userapi-new" v-click-outside="clickOutside">
                <button class="userapi-new-btn" @click="toggleUserAPISelector(true)">{{$t("CustomQuery['新增条件']")}}</button>
                <div class="userapi-new-selector-wrapper" v-show="isPropertiesShow">
                    <bk-select class="userapi-new-selector" 
                        :selected.sync="selectedObjId">
                        <bk-select-option v-for="(obj, index) in object"
                            :key="index"
                            :value="obj.id"
                            :label="obj.name">
                        </bk-select-option>
                    </bk-select>   
                    <bk-select class="userapi-new-select"
                        ref="propertySelector"
                        @on-selected="addUserProperties">
                            <!-- <bk-select-option v-for="(property, index) in filterProperty(object[selectedObjId]['properties'])" -->
                            <bk-select-option v-for="(property, index) in object[selectedObjId]['properties']"
                                :disabled="property.disabled"
                                :key="property['bk_property_id']"
                                :value="property['bk_property_id']"
                                :label="property['bk_property_name']">
                            </bk-select-option>
                    </bk-select>
                </div>
            </div>
        </div>
        <div class="userapi-btn-group">
            <bk-button type="primary" class="userapi-btn" :disabled="errors.any()" @click.stop="previewUserAPI">
                {{$t("CustomQuery['预览']")}}
            </bk-button>
            <bk-button type="primary" class="userapi-btn" :disabled="errors.any()" @click="saveUserAPI">
                {{$t("Common['保存']")}}
            </bk-button>
            <bk-button type="default" class="userapi-btn vice-btn" @click="closeSlider">
                {{$t("Common['取消']")}}
            </bk-button>
            <bk-button type="default" class="userapi-btn del-btn" @click="deleteUserAPIConfirm" v-if="type === 'update'">
                {{$t("Common['删除']")}}
            </bk-button>
        </div>
        <v-preview :isPreviewShow.sync="isPreviewShow" :apiParams="apiParams" :attribute="object"></v-preview>
    </div>
</template>
<script>
    import vOperator from './operator'
    import vApplicationSelector from '@/components/common/selector/application'
    import vValidate from '@/components/common/validator/validate'
    import vPreview from './preview'
    import { mapGetters } from 'vuex'
    export default {
        props: {
            bkBizId: {
                required: true      // 当前业务ID
            },
            id: {
                default: ''
            },
            type: {
                type: String,
                default: 'create'   // 当前定义类型是新增('create')还是修改('update')
            },
            isShow: {
                type: Boolean,      // 侧滑栏是否显示，watch此参数便于清空内容
                required: true
            }
        },
        data () {
            return {
                name: '',
                attribute: {
                    list: [],
                    selected: '',
                    default: [{
                        'bk_property_id': 'bk_host_innerip',
                        'bk_property_name': this.$t("Common['内网IP']"),
                        'disabled': true
                    }, {
                        'bk_property_id': 'bk_biz_name',
                        'bk_property_name': this.$t("Common['业务']"),
                        'disabled': true
                    }, {
                        'bk_property_id': 'bk_set_name',
                        'bk_property_name': this.$t("Hosts['集群']"),
                        'disabled': true
                    }, {
                        'bk_property_id': 'bk_module_name',
                        'bk_property_name': this.$t("Hosts['模块']"),
                        'disabled': true
                    }, {
                        'bk_property_id': 'bk_cloud_id',
                        'bk_property_name': this.$t("Hosts['云区域']"),
                        'disabled': true
                    }]
                },
                userProperties: [], // 自定义查询条件
                dataCopy: {
                    name: '',
                    userProperties: [],
                    attributeSelected: ''
                },
                isPropertiesShow: false, // 自定义条件下拉列表展示与否
                isPreviewShow: false, // 显示预览
                object: {
                    'host': {
                        id: 'host',
                        name: this.$t("Hosts['主机']"),
                        properties: []
                    },
                    'set': {
                        id: 'set',
                        name: this.$t("Hosts['集群']"),
                        properties: []
                    },
                    'module': {
                        id: 'module',
                        name: this.$t("Hosts['模块']"),
                        properties: []
                    }
                },
                selectedObjId: 'host',
                operatorMap: {
                    'time': '$in',
                    'enum': '$eq'
                }
            }
        },
        computed: {
            ...mapGetters(['bkSupplierAccount']),
            /* 生成保存自定义API的参数 */
            apiParams () {
                let paramsMap = [
                    {'bk_obj_id': 'set', condition: [], fields: []},
                    {'bk_obj_id': 'module', condition: [], fields: []},
                    {
                        'bk_obj_id': 'biz',
                        condition: [{
                            field: 'default', // 该参数表明查询非资源池下的主机
                            operator: '$ne',
                            value: 1
                        }],
                        fields: []
                    }, {
                        'bk_obj_id': 'host',
                        condition: [],
                        fields: this.attribute.selected ? this.attribute.selected.split(',') : []
                    }
                ]
                const specialObj = {
                    'host': 'bk_host_innerip',
                    'biz': 'bk_biz_name',
                    'plat': 'bk_cloud_name',
                    'module': 'bk_module_name',
                    'set': 'bk_set_name'
                }
                this.userProperties.forEach((property, index) => {
                    let param = paramsMap.find(({bk_obj_id: bkObjId}) => {
                        return bkObjId === property.bkObjId
                    })
                    if (property.bkPropertyType === 'singleasst' || property.bkPropertyType === 'multiasst') {
                        paramsMap.push({
                            'bk_obj_id': property.bkAsstObjId,
                            fields: [],
                            condition: [{
                                field: specialObj.hasOwnProperty(property.bkAsstObjId) ? specialObj[property.bkAsstObjId] : 'bk_inst_name',
                                operator: property.operator,
                                value: property.value
                            }]
                        })
                    } else if (property.bkPropertyType === 'time' || property.bkPropertyType === 'date') {
                        let value = property['value'].split(' - ')
                        param['condition'].push({
                            field: property.bkPropertyId,
                            operator: value[0] === value[1] ? '$eq' : '$gte',
                            value: value[0]
                        })
                        param['condition'].push({
                            field: property.bkPropertyId,
                            operator: value[0] === value[1] ? '$eq' : '$lte',
                            value: value[1]
                        })
                    } else if (property.bkPropertyType === 'bool' && ['true', 'false'].includes(property.value)) {
                        param['condition'].push({
                            field: property.bkPropertyId,
                            operator: property.operator,
                            value: property.value === 'true'
                        })
                    } else {
                        let operator = property.operator
                        let value = property.value
                        // 多模块与多集群查询
                        if (property.bkPropertyId === 'bk_module_name' || property.bkPropertyId === 'bk_set_name') {
                            operator = operator === '$regex' ? '$in' : operator
                            if (operator === '$in') {
                                let arr = value.replace('，', ',').split(',')
                                let isExist = arr.findIndex(val => {
                                    return val === value
                                }) > -1
                                value = isExist ? arr : [...arr, value]
                            }
                        }
                        param['condition'].push({
                            field: property.bkPropertyId,
                            operator: operator,
                            value: value
                        })
                    }
                })
                let params = {
                    'bk_biz_id': this.bkBizId,
                    'info': {
                        condition: paramsMap
                    },
                    'name': this.name
                }
                if (this.type === 'update') {
                    params['id'] = this.id
                }
                return params
            }
        },
        watch: {
            /* 监听侧滑栏的显示状态，显示则初始化相关下拉列表，不显示则清空内容 */
            isShow (isShow) {
                if (!isShow) {
                    setTimeout(() => {
                        this.resetDefine()
                        this.$validator.validateAll().then(() => {
                            this.errors.clear()
                        })
                    })
                } else if (this.id) {
                    this.getUserAPIDetail()
                }
            },
            'object.host.properties' (properties) {
                let selected = []
                let tempList = []
                properties.map(property => {
                    let isDefaultPropery = false
                    selected = this.attribute.default.map(defaultProperty => {
                        if (property['bk_property_id'] === defaultProperty['bk_property_id']) {
                            isDefaultPropery = true
                        }
                        return defaultProperty['bk_property_id']
                    })
                    if (!isDefaultPropery) {
                        tempList.push(property)
                    }
                })
                this.attribute.list = tempList.concat(this.attribute.default)
                this.attribute.selected = selected.join(',')
                this.dataCopy.attributeSelected = this.attribute.selected
            },
            selectedObjId () {
                this.$refs.propertySelector.$forceUpdate()
            }
        },
        mounted () {
            this.initObjectProperties()
        },
        methods: {
            isCloseConfirmShow () {
                if (this.name !== this.dataCopy.name || this.dataCopy.attributeSelected !== this.attribute.selected || this.userProperties.length !== this.dataCopy.userProperties.length) {
                    return true
                }
                for (let i = 0; i < this.userProperties.length; i++) {
                    let property = this.userProperties[i]
                    let propertyCopy = this.dataCopy.userProperties[i]
                    for (let key in property) {
                        if (property[key] !== propertyCopy[key]) {
                            return true
                        }
                    }
                }
                return false
            },
            deleteUserAPIConfirm () {
                this.$bkInfo({
                    title: this.$t("CustomQuery['确认要删除']", {name: this.apiParams.name}),
                    confirmFn: () => {
                        this.deleteUserAPI()
                    }
                })
            },
            /*
                删除自定义API
            */
            async deleteUserAPI () {
                try {
                    await this.$axios.delete(`userapi/${this.bkBizId}/${this.id}`)
                    this.$emit('delete')
                    this.$emit('cancel')
                    this.$alertMsg(this.$t("Common['删除成功']"), 'success')
                } catch (e) {
                    this.$alertMsg(e.data['bk_error_msg'])
                }
            },
            initObjectProperties () {
                this.$Axios.all([this.getObjectProperty('host'), this.getObjectProperty('set'), this.getObjectProperty('module')])
                .then(this.$Axios.spread((hostRes, setRes, moduleRes) => {
                    this.object['host']['properties'] = (hostRes.result ? hostRes.data : []).filter(property => !property['bk_isapi'])
                    this.object['set']['properties'] = (setRes.result ? setRes.data : []).filter(property => !property['bk_isapi'])
                    this.object['module']['properties'] = (moduleRes.result ? moduleRes.data : []).filter(property => !property['bk_isapi'])
                    this.addDisabled()
                }))
            },
            getObjectProperty (bkObjId) {
                return this.$axios.post('object/attr/search', {
                    'bk_obj_id': bkObjId,
                    'bk_supplier_account': this.bkSupplierAccount
                }).then((res) => {
                    if (!res.result) {
                        this.$alertMsg(res['bk_error_msg'])
                    }
                    return res
                })
            },
            getUserAPIDetail () {
                this.$axios.get(`userapi/detail/${this.bkBizId}/${this.id}`).then(res => {
                    if (res.result) {
                        this.setUserProperties(res.data)
                    } else {
                        this.$alertMsg(res['bk_error_msg'])
                    }
                })
            },
            addDisabled () {
                for (let objId in this.object) {
                    this.object[objId]['properties'].map((property) => {
                        property.disabled = false
                    })
                }
            },
            filterProperty (properties) {
                return properties.filter(property => {
                    let {
                        bk_isapi: bkIsapi,
                        bk_property_type: bkPropertyType
                    } = property
                    return !bkIsapi && bkPropertyType !== 'multiasst' && bkPropertyType !== 'singleasst'
                })
            },
            setUserProperties (detail) {
                let properties = []
                let info = JSON.parse(detail['info'])
                info.condition.forEach(condition => {
                    condition['condition'].forEach(property => {
                        let originalProperty = this.getOriginalProperty(property.field, condition['bk_obj_id'])
                        if (originalProperty) {
                            if (['time', 'date'].includes(originalProperty['bk_property_type']) && properties.some(({bkPropertyId}) => bkPropertyId === originalProperty['bk_property_id'])) {
                                let repeatProperty = properties.find(({bkPropertyId}) => bkPropertyId === originalProperty['bk_property_id'])
                                repeatProperty.value = [repeatProperty.value, property.value].join(' - ')
                            } else {
                                properties.push({
                                    'bkObjId': originalProperty['bk_obj_id'],
                                    'bkPropertyType': originalProperty['bk_property_type'],
                                    'bkPropertyName': originalProperty['bk_property_name'],
                                    'bkPropertyId': originalProperty['bk_property_id'],
                                    'bkAsstObjId': originalProperty['bk_asst_obj_id'],
                                    'operator': property.operator,
                                    'value': property.value
                                })
                            }
                            originalProperty.disabled = true
                        }
                    })
                    if (condition['bk_obj_id'] === 'host') {
                        this.attribute.selected = condition['fields'].join(',')
                    }
                })
                this.userProperties = properties
                this.name = detail['name']
                this.dataCopy = {
                    name: detail['name'],
                    userProperties: this.$deepClone(properties),
                    attributeSelected: this.attribute.selected
                }
            },
            addUserProperties ({value: bkPropertyId}, index) {
                let property = this.getOriginalProperty(bkPropertyId, this.selectedObjId)
                let {
                    'bk_property_name': bkPropertyName,
                    'bk_property_type': bkPropertyType,
                    'bk_asst_obj_id': bkAsstObjId,
                    'bk_obj_id': bkObjId
                } = property
                property.disabled = true
                this.userProperties.push({
                    bkObjId,
                    bkPropertyId,
                    bkPropertyType,
                    bkPropertyName,
                    bkAsstObjId,
                    operator: this.operatorMap.hasOwnProperty(bkPropertyType) ? this.operatorMap[bkPropertyType] : '',
                    value: ''
                })
                this.$refs.propertySelector.curLabel = ''
                this.$refs.propertySelector.curValue = ''
                this.toggleUserAPISelector(false)
            },
            setUserPropertyTime (oldTime, newTime, index) {
                this.userProperties[index]['value'] = newTime
            },
            /* 通过选择的propertyId, 查找其对应的对象，以获得更多信息 */
            getOriginalProperty (bkPropertyId, bkObjId) {
                let property = null
                for (let objId in this.object) {
                    for (var i = 0; i < this.object[objId]['properties'].length; i++) {
                        let loopProperty = this.object[objId]['properties'][i]
                        if (loopProperty['bk_property_id'] === bkPropertyId && loopProperty['bk_obj_id'] === bkObjId) {
                            property = loopProperty
                            break
                        }
                    }
                    if (property) {
                        break
                    }
                }
                return property
            },
            getEnumOptions (userProperty) {
                let property = this.getOriginalProperty(userProperty.bkPropertyId, userProperty.bkObjId)
                if (property) {
                    return property.option || []
                }
                return []
            },
            /* 删除自定义条件时，恢复下拉列表中对应的项为可点击状态 */
            deleteUserProperty (userProperty, index) {
                let property = this.getOriginalProperty(userProperty.bkPropertyId, userProperty.bkObjId)
                property.disabled = false
                this.userProperties.splice(index, 1)
            },
            /* 切换新增条件的显示 */
            toggleUserAPISelector (isPropertiesShow) {
                this.isPropertiesShow = isPropertiesShow
            },
            clickOutside () {
                this.toggleUserAPISelector(false)
            },
            /* 侧滑栏关闭时，重置自定义编辑界面的内容 */
            resetDefine () {
                this.isPropertiesShow = false
                this.isPreviewShow = false
                this.name = ''
                this.userProperties = []
                this.attribute.selected = this.attribute.default.map(({bk_property_id: bkPropertyId}) => bkPropertyId).join(',')
                Object.keys(this.object).map(bkObjId => {
                    this.object[bkObjId]['properties'].map(property => {
                        property.disabled = false
                    })
                })
            },
            /* 保存自定义条件 */
            saveUserAPI () {
                this.$validator.validateAll().then(isValid => {
                    if (isValid) {
                        // 将Info字段转为JSON字符串提交
                        let params = Object.assign({}, this.apiParams, {'info': JSON.stringify(this.apiParams['info'])})
                        if (this.type === 'create') {
                            this.$axios.post('userapi', params).then(res => {
                                if (res.result) {
                                    this.$alertMsg(this.$t("Common['保存成功']"), 'success')
                                    this.$emit('create', res.data)
                                } else {
                                    this.$alertMsg(res['bk_error_msg'])
                                }
                            })
                        } else {
                            this.$axios.put(`userapi/${this.bkBizId}/${this.id}`, params)
                            .then(res => {
                                if (res.result) {
                                    this.$emit('update', res.data)
                                    this.$alertMsg(this.$t("Common['修改成功']"), 'success')
                                } else {
                                    this.$alertMsg(res['bk_error_msg'])
                                }
                            })
                        }
                    }
                })
            },
            previewUserAPI () {
                this.$validator.validateAll().then(isValid => {
                    if (isValid) {
                        this.isPreviewShow = true
                    } else {
                        this.$alertMsg(this.errors.all()[0])
                    }
                })
            },
            /* 关闭侧滑层 */
            closeSlider () {
                this.$emit('cancel')
            }
        },
        components: {
            vOperator,
            vApplicationSelector,
            vPreview,
            vValidate
        }
    }
</script>
<style lang="scss" scoped>
    .userapi-wrapper{
        padding: 20px 40px;
        height: calc(100% - 60px);
        overflow: hidden;
        overflow-y: auto;
    }
    .userapi-group{
        margin-top: 20px;
    }
    .userapi-list{
        line-height: 30px;
        color: #737987;
        .userapi-item{
            margin-top: 20px;
            position: relative;
            .userapi-name{
                width: 115px;
                line-height: 32px;
                padding-right: 15px;
                text-align: right;
            }
            .userapi-text{
                position: relative;
                width: 369px;
                height: 32px;
                padding: 0 8px;
                margin: 0 5px 0 -1px;
                border-radius: 2px;
                border-top-left-radius: initial;
                border-bottom-left-radius: initial;
                &:focus{
                    z-index: 2;
                }
            }
            .userapi-delete{
                display: block;
                width: 30px;
                height: 32px;
                margin: 0 5px;
                line-height: 30px;
                text-align: center;
                border: 1px solid #bec6de;
                border-radius: 2px;
                cursor: pointer;
                &:hover{
                    border-color: #ef4c4c;
                }
            }
        }
    }
    .userapi-new{
        width: 470px;
        margin: 20px 0 0 120px;
        font-size: 14px;
        // color: #6b7baa;
        position: relative;
        .userapi-new-btn{
            width: 470px;
            height: 42px;
            background-color: #ffffff;
            border-radius: 2px;
            border: solid 1px #c3cdd7;
            outline: 0;
            &:hover{
                box-shadow: 0px 3px 6px 0px rgba(51, 60, 72, 0.1);
            }
        }
    }
    .userapi-new-selector-wrapper{
        position: absolute;
        top: 100%;
        left: 0;
        width: 470px;
        margin-top: 3px;
        background-color: #ffffff;
        box-shadow: 0px 3px 6px 0px rgba(51, 60, 72, 0.1);
        border-radius: 2px;
        border: solid 1px #bec6de;
        z-index: 10;
    }
    .userapi-input{
        margin-top: 26px;
        position: relative;
        .userapi-input-name{
            width: 115px;
            line-height: 32px;
            text-align: right;
            padding-right: 15px;
        }
        .userapi-input-text{
            width: 470px;
            height: 32px;
            // border-radius: 2px;
            // // border: solid 1px #bec6de;
            margin: 0 5px;
            // padding: 0 8px;
        }
    }
    .userapi-btn-group{
        margin: 40px 0 0 120px;
        font-size: 0;
        .userapi-btn{
            width: 110px;
            height: 34px;
            margin: 0 10px 0 0;
            font-size: 14px;
            // letter-spacing: 4px;
        }
    }
    .validate-message{
        position: absolute;
        top: 100%;
        left: 120px;
        height: 16px;
        line-height: 16px;
    }
</style>
<style lang="scss">
    .bk-selector,
    .bk-select{
        &.userapi-compare-selector,
        &.userapi-new-selector,
        &.userapi-content-selector{
            .bk-selector-input{
                height: 32px;
                line-height: 30px;
                border-top-right-radius: initial;
                border-bottom-right-radius: initial;
            }
            .bk-selector-icon{
                top: 50%;
                transform: translateY(-50%);
            }
        }
        &.userapi-compare-selector.open,
        &.userapi-new-selector.open{
            .bk-selector-icon{
                transform: translateY(-50%) rotate(180deg);
            }
        }
        &.userapi-compare-selector{
            width: 102px;
            margin: 0 0 0 5px;
            z-index: 1;
            &.open{
                z-index: 3;
            }
        }
        &.userapi-new-selector{
            width: 445px;
            margin: 6px auto 0;
        }
        &.userapi-content-selector{
            width: 470px;
            margin: 0 5px;
            .bk-select-input{
                height: 32px;
                line-height: 32px;
                padding: 0 32px 0 8px;
            }
        }
    }
    .bk-select{
        &.userapi-new-select{
            .bk-select-wrapper{
                display: none;
            }
            .bk-select-list{
                display: block !important;
                position: static;
                margin-top: 5px;
                box-shadow: none;
                & > ul {
                    max-height: 205px;
                }
            }
            .bk-select-group-name{
                border: none;
                padding: 8px 20px;
            }
        }
    }
    .bk-date.userapi-date{
        width: 470px !important;
        margin: 0 5px;
        height: 32px;
        line-height: 32px;
        z-index: 2;
        &:after{
            width: 32px;
            height: 32px;
        }
        [name="date-select"]{
            height: 32px;
            line-height: 32px;
            border-radius: 2px;
        }
    }
    .bk-select.userapi-enum{
        width: 470px;
        margin: 0 5px;
        height: 32px;
        line-height: 32px;
        .bk-select-input{
            height: 32px;
            line-height: 32px;
        }
    }
    .userapi-item.form-validate-error{
        .bk-selector-wrapper{
            .bk-selector-input{
                border-color: #ff5656;
            }
        }
        .userapi-text{
            border-color: #ff5656;
        }
    }
</style>