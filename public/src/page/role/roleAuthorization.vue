<template>
    <div class="tab-content">
        <el-select
                v-model="value"
                :multiple="false"
                filterable
                remote
                reserve-keyword
                placeholder="请输入商户关键词"
                :remote-method="remoteMethod"
                :focus="GetFocus"
                :loading="loading"
        style="margin-bottom:20px;">
            <el-option
                    v-for="item in options"
                    :key="item.uuid"
                    :label="item.name"
                    :value="item.uuid">
            </el-option>
        </el-select>
        <el-button type="primary" icon="plus" @click="AddingMerchant"></el-button>
        <el-button :plain="true" type="info" @click='blackPage' style="display:block;float:right;margin-right: 20px;">返回</el-button>
        <div>
            <el-button style="margin:10px 10px;" v-for="tag in BusinessListArray" @click="BusinessClick(tag)" type="primary" round>{{tag.name}}</el-button>
        </div>
        <el-table :data="BusinesstableData" border style="width: 100%">
            <el-table-column label="商户名称" width="200">
                <template scope="scope">
                    <span style="margin-left: 10px">{{ scope.row.name }}</span>
                </template>
            </el-table-column>
            <el-table-column label="权限" style="text-align: left">
                <template scope="scope">
                    <ul class="tab_box">
                        <li v-for="list in scope.row.children">
                            <div class="tab_left">
                                <el-checkbox-group  v-model="checkAll" @change="handleCheckAllChange">
                                    <el-checkbox :label="list.label" :key="list.id">{{list.label}}</el-checkbox>
                                </el-checkbox-group>
                            </div>
                            <div class="tab_right">
                                <el-checkbox-group  v-model="checkedCities" @change="handleCheckedCitiesChange">
                                    <el-checkbox v-for="listTwo in list.children"  :label="listTwo.label" :key="listTwo.id">{{listTwo.label}}</el-checkbox>
                                </el-checkbox-group>
                            </div>
                        </li>
                    </ul>
                </template>
            </el-table-column>
            <el-table-column label="操作" width="180">
                <template scope="scope">
                    <el-button size="small" type="info" @click="handleSeeEdit(scope.$index, scope.row)">保存</el-button>
                </template>
            </el-table-column>
        </el-table>
    </div>
</template>
<script>
    import {backendPrivilegeList,backendPrivilegeParent,businessSearch,rolePrivilegeBusiness,RolePrivilegeAdd,RolePrivilegeBusinessRole} from './../../../api/api';
    export default {
        props: {
        },
        data() {
            return {
                activeName: 'see',
                BusinesstableData: [],//展示权限数据
                //选择
                PermissionsListArray:[],//权限列表替换
                options: [],
                value:'',
                list: [],
                loading: false,

                //选择权限
                isIndeterminate:false,
                checkAll: [],
                checkedCities:[],
                //商户列表
                BusinessListArray:[],
                ids:[]//已选权限数组
            };
        },
        mounted() {},
        watch: {
            filterText(val) {
                this.$refs.tree2.filter(val);
            }
        },
        created: function () {
            this.roleBusiness();//获取商户
            this.PermissionsList();//获取权限
        },
        methods: {
            handleClick(tab, event) {
                console.log(tab, event);
            },
            blackPage(){
                this.$router.go(-1);
            },
            handleSeeEdit(index, row) {//保存商户与角色绑定
//                console.log(index, row);
//                console.log(row.business_uuid);
                let idArray=[];
                console.log(this.checkAll);
                if(this.checkAll.length>0){
                    for(let i=0;i<this.checkAll.length;i++){
                        for(let j=0;j<this.PermissionsListArray.length;j++){
                            if(this.checkAll[i]==this.PermissionsListArray[j].label){
                                idArray.push(this.PermissionsListArray[j].id);
                                for(let m=0;m<this.PermissionsListArray[j].children.length;m++){
                                    for(let n=0;n<this.checkedCities.length;n++){
                                        if(this.checkedCities[n]=this.PermissionsListArray[j].children[m].label){
                                            idArray.push(this.PermissionsListArray[j].children[m].id);
                                        }
                                    }
                                }
                            }
                        }
                    }
                    //js数组去除重复
                    for(let p in idArray){
                        if(this.ids.indexOf(idArray[p])==-1){
                            this.ids.push(idArray[p]);
                        }
                    }
                    this.RoleBusinessAdd(row.business_uuid)
                }else{
                    this.$notify({
                        title: '警告',
                        message: '请选择权限',
                        type: 'warning'
                    });
                }
                console.log(this.ids.length);
            },
            RoleBusinessAdd(business_uuid){//保存角色和商户权限绑定
                let localRoutes =JSON.parse(window.sessionStorage.getItem('user'));
                let idslistArray=this.ids.join(",");
                console.log(idslistArray);
                let AddSearch={
                    access_token:localRoutes.access_token,
                    role_id:this.$route.query.role_id,
                    business_uuid:business_uuid,
                    ids:idslistArray
                };
                RolePrivilegeAdd(AddSearch).then(data=>{
                    console.log(data);
                    if(data.code==0){
                        this.$notify({
                            title: '成功',
                            message: '保存成功',
                            type: 'success'
                        });
                    }else{
                        this.$notify.error({
                            title: '错误提示',
                            message: data.message
                        });
                    }
                });
            },
            handleSeeDelete(index, row) {
                console.log(index, row);
            },
            handleClose(row,tag){
                console.log(row);
                console.log(tag);
                for(var i=0;i<this.BusinesstableData.length;i++){
                    if(this.tableData[i].id==row.id){
                        console.log(tag.id);
                        for(var j=0;j<this.BusinesstableData[i].children.length;j++){
                            if(this.BusinesstableData[i].children[j].id==tag.id){
                                this.BusinesstableData[i].children.splice(j,1);
                                return;
                            }
                        }
//                        this.tableData[i].children.splice(this.tableData[i].children.indexOf(tag.id), 1);
                    }
                }
            },
            roleBusiness(){//查看已绑定商户
                let localRoutes =JSON.parse(window.sessionStorage.getItem('user'));
                let ROLESearch={
                    access_token:localRoutes.access_token,
                    role_id:this.$route.query.role_id
                };
                rolePrivilegeBusiness(ROLESearch).then(data=>{
                    console.log(data);
                    if(data.code==0){
                        for(let i=0;i<data.content.length;i++){
                            this.BusinessListArray.push(data.content[i]);
                        }
                    }else{
                        this.$notify.error({
                            title: '错误提示',
                            message: data.message
                        });
                    }
                });
            },
            remoteMethod(query) {
                if (query !== '') {
                    this.loading = true;
                    let localRoutes =JSON.parse(window.sessionStorage.getItem('user'));
                    let MerchantSearch={
                        access_token:localRoutes.access_token,
                        name:query
                    };
                    businessSearch(MerchantSearch).then(data=>{
                        this.loading = false;
                        if(data.code==0){
                            this.options =data.content;
                            if(this.options.length==0){
                                this.value='';
                            }
                        }else{
                            this.$notify.error({
                                title: '错误提示',
                                message: data.message
                            });
                        }
                    });
                } else {
                    this.options = [];
                    this.value='';
                }
            },
            GetFocus(){
                this.options = [];
                this.value='';
            },
            AddingMerchant(){//添加商户权限
                if(this.options.length==0){
                    this.$notify({
                        title: '警告',
                        message: '请输入搜索信息',
                        type: 'warning'
                    });
                }else{
                    if(this.BusinessListArray.length!=0){
                        let TmpArray=[];
                        for(let i=0;i<this.BusinessListArray.length;i++){
                            TmpArray.push(this.BusinessListArray[i].name);
                        }
                        let numTmp=TmpArray.indexOf (this.options[0].name);
                        console.log(numTmp);
                        if(numTmp!=-1){
                            this.$notify({
                                title: '警告',
                                message: '商户已存在！',
                                type: 'warning'
                            });
                        }else{
                            let MerchantAuthorityList={
                                name:this.options[0].name,
                                business_uuid:this.options[0].uuid,
                            };
                            this.BusinessListArray.push(MerchantAuthorityList);
                        }
                    }else{
                        let MerchantAuthorityList={
                            name:this.options[0].name,
                            business_uuid:this.options[0].uuid,
                        };
                        this.BusinessListArray.push(MerchantAuthorityList);
                    }
                }
            },
            PermissionsList(){//获取权限子列表
                let localRoutes =JSON.parse(window.sessionStorage.getItem('user'));
                    var leftParams = {
                        access_token:localRoutes.access_token
                    };
                    backendPrivilegeList(leftParams).then(data=>{
                        this.PermissionsListArray=[];
                        if(data.code==0){
                            for(let i=0;i<data.content.length;i++){
                                let Parents={
                                    access_token:localRoutes.access_token,
                                    parent_id:data.content[i].id
                                };
                                let list={
                                    id:data.content[i].id,
                                    label:data.content[i].name,
                                    children:[],
                                    isLeaf: 'leaf'
                                };
                                backendPrivilegeParent(Parents).then(data=>{
                                    if(data.code==0){
                                        for(let j=0;j<data.content.length;j++){
                                              let son={
                                                  id:data.content[j].id,
                                                  label:data.content[j].name
                                              };
                                              list.children.push(son)
                                        }
                                    }
                                });
                                this.PermissionsListArray.push(list)
                            }
                        }else{
                            this.$notify.error({
                                title: '错误提示',
                                message: data.message
                            });
                        }
                    });
            },
            handleCheckAllChange(value) {//父级权限
                this.checkedCities=[];
                if(value.length>0){
                    for(let i=0;i<value.length;i++){
                        for(let j=0;j<this.PermissionsListArray.length;j++){
                            if(value[i]==this.PermissionsListArray[j].label){
                                for(let m=0;m<this.PermissionsListArray[j].children.length;m++){
                                    this.checkedCities.push(this.PermissionsListArray[j].children[m].label);
                                }
                            }
                        }
                    }
                }

            },
            handleCheckedCitiesChange(value) {//子权限
                    for(let j=0;j<this.PermissionsListArray.length;j++){
                        if(this.PermissionsListArray[j].children.length!=0){
                            for(let o= 0; o <this.checkAll.length; o++){
                                if(this.checkAll[o]==this.PermissionsListArray[j].label){
                                    this.checkAll.splice(o,1);
                                    o--;
                                }
                            }
                        }
                    }
                    this.ArrayAddition(value);
            },
            ArrayAddition(value){//与上面协同合作
                if(value.length>0){
                    for(let i=0;i<value.length;i++){
                        for(let j=0;j<this.PermissionsListArray.length;j++){
                            if(this.PermissionsListArray[j].children.length!=0){
                                for(let m=0;m<this.PermissionsListArray[j].children.length;m++){
                                    if(value[i]==this.PermissionsListArray[j].children[m].label){
                                        this.checkAll.push(this.PermissionsListArray[j].label);
                                    }
                                }
                            }
                        }
                    }
                    //js数组去除重复
                    for(let p in this.checkAll){
                        if(this.checkAll.indexOf(this.checkAll[p])==-1){
                            this.checkAll.push(this.checkAll[p]);
                        }
                    }
                }
            },
            BusinessClick(tag){//商户点击
                console.log(tag);
                this.BusinesstableData=[];
                this.checkAll=[];
                this.checkedCities=[];
                let roleArray={
                    "name":tag.name,
                    "business_uuid":tag.business_uuid,
                    "children":this.PermissionsListArray
                };
                let localRoutes =JSON.parse(window.sessionStorage.getItem('user'));
                let nessSearch={
                    access_token:localRoutes.access_token,
                    role_id:this.$route.query.role_id,
                    business_uuid:tag.business_uuid
                };
                RolePrivilegeBusinessRole(nessSearch).then(data=>{
                    if(data.code==0){
                        this.BusinesstableData.push(roleArray);
                        if(data.content!=''){
                            for(let i=0;i<data.content.length;i++){
                                this.checkAll.push(data.content[i].privilege_name);
                                if(data.content[i].children!=undefined){
                                    for(let j=0;j<data.content[i].children.length;j++){
                                        this.checkedCities.push(data.content[i].children[j].privilege_name);
                                    }
                                }
                            }
                        }else{

                        }
                    }else{
                        this.$notify.error({
                            title: '错误提示',
                            message: data.message
                        });
                    }
                });
            },

        }
    };
</script>

<style lang="css">
    /*居中*/
    ul,li {list-style-type:none;margin:0;padding:0}
    .cell {
        text-align: center;
    }
    .tab-content{
        margin-top: 10px;
    }
    .VisibleType{
        cursor:pointer;
        margin:0 10px;
    }
    .VisibleType:hover{
        background-color:#3390FF;
        color:#fff;
    }
    .tab_box li{
        height:50px;
        width:100%;
        border-bottom:1px solid #DFE6EC;
    }
    .tab_box li div{
        /*position: absolute;*/
        float: left;
        display: block;
    }
    .tab_left{
        width:20%;
        height:50px;
        text-align: left;
        line-height: 50px;
        border-right:1px solid #DFE6EC;
    }
    .tab_right{
        width:70%;
        height:50px;
        padding-left:5%;
        text-align: left;
        line-height: 50px;
    }
</style>