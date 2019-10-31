<template>
  <div>
    <div class="dialogDiv" v-loading="OrgLoading">
      <ul>
        <li>
          <el-input placeholder="输入关键字进行筛选" v-model="filterText" clearable></el-input>
          <el-tree :data="orgList" node-key="id" highlight-current  ref="treeBox" :props="propsTree" :expand-on-click-node="false" :filter-node-method="filterNode" @node-click="nodeClick"></el-tree>
        </li>
        <li v-loading="UserLoading">
          <el-input placeholder="输入人员名称进行筛选" v-model="filterPerson" clearable></el-input>
          <div v-if="userList.length>0" style="margin-top: 5px;text-align: left;">
            <el-checkbox :indeterminate="isIndeterminate" v-model="checkAll" @change="CheckAllPerson">全选</el-checkbox>
            <span>（会议人员）</span>
            <div style="margin-top:15px;margin-left:20px;">
              <el-checkbox-group v-model="cities" @change="RadioPerson" class="dialigGroup">
                  <span style="display:block;box-sizing: border-box;padding: 5px" v-for="item in userList" :key="item.id">
                    <el-checkbox :label="item.id+'-'+item.name">{{item.name}}</el-checkbox>
                  </span>
              </el-checkbox-group>
            </div>
          </div>
          <div v-else style="color:red;margin:30px auto!important;font-size: 14px;width: 100px">暂无数据！</div>
        </li>
      </ul>
    </div>
    <el-button type="primary" size="small"  @click="outcome" style="margin-top: 20px;" >任务派发对象名单</el-button>
  </div>
</template>

<script>
import { getAllOrg } from '@/api/bulletinBoard'; // 部门列表
import { getAllPage } from "@/api/departmental/departmentalPersonnel"; // 对应部门人员列表
import { getTokenMap } from "@/utils/auth";
export default {
  props: {
    array:{
      type:Array,
      default: ()=>{
        return []
      }
    },
  },
  data() {
    return {
      OrgLoading:true,
      UserLoading:true,
      filterText: null, // 过滤部门
      orgList:[],// 部门列表
      orgObj:{ // 当前点击的部门对象
        id:undefined,
        label:undefined
      },
      propsTree: { // tree组件配置项
        children: 'children',
        label: 'label'
      },
      filterPerson:null, // 过滤人员
      isIndeterminate: false, // 半选状态，ture:为半选
      checkAll: false,//全选标识
      userList: [],//点击部门后获取的部门人员列表
      userMap:[],//点击部门后获取的部门人员列表,克隆一份，目的是在筛选过滤人员时用到
      cities: [],//多选框选中的人员列表
      personRoster:[], // 最终选中的人员列表
    }
  },
  watch: {
    array:{
      handler(val, oldVal) {
        this.personRoster = [];
        for(let v of val){
          let obj = v.id+"-"+v.name;
          this.personRoster.push(obj);
        }
        this.getAllPage();
      },
      immediate: true,
    },
    filterText(val) { // 过滤部门
      this.$refs.treeBox.filter(val);
    },
    filterPerson(val) { // 过滤人员
      this.checkAll = false  //初始化全选状态
      this.isIndeterminate = false  //初始化全选状态
      if(val){
        this.userList = this.userMap.filter((item)=>{
          if (!item) return true;
          return item.name.indexOf(val) !== -1; // 注意：这里的label要与 propsTree里的配label字段相同
        });
      }else{
        this.userList = this.userMap;
      }
      let arr = [];
      for(let x of this.userList){
        let obj = x.id+"-"+x.name;
        for(let v of this.cities){
          if(obj == v){
            arr.push(v);
            break;
          }
        }
      }
      this.checkAll = arr.length === this.userList.length;
      this.isIndeterminate = arr.length > 0 && arr.length < this.userList.length;
    }
  },
  created(){
    this.getAllOrg();
  },
  mounted(){
  },
  computed: {

  },
  methods: {
    // 获取部门列表树
    async getAllOrg() {
      this.OrgLoading = true;
      let res = await getAllOrg();
      if (res.code == 1) {
        this.orgList = res.data;
        this.orgObj.id = res.data[0].id;
        this.orgObj.label = res.data[0].label;
        this.$nextTick(() => {
          this.$refs.treeBox.setCurrentKey(this.orgObj.id);
        });
        this.getAllPage();
      }else{
        this.$message({ type: 'error', message: res.msg});
      }
      this.OrgLoading = false;
    },
    // 获取对应部门人员列表
    async getAllPage() {
      this.UserLoading = true;
      let res = await getAllPage({page: 1,limit: 100,organId: this.orgObj.id,});
      if (res.code == 1) {
        this.userList = res.data.content;
        this.userMap = res.data.content;
        this.handlePerson();
      }else{
        this.$message({ type: 'error', message: res.msg});
      }
      this.UserLoading = false;
    },
    // 处理人员名单-判断该部门的人员列表中哪些人员已经选择了
    handlePerson(){
      this.cities = [];
      this.checkAll = false;
      this.isIndeterminate = false;
      for(let obj of this.personRoster){
        let id = obj.split("-")[0]; // 这里obj是为id+'-'+name的字符串
        for(let v in this.userList){
          if(id == this.userList[v].id){
            let string = this.userList[v].id+"-"+this.userList[v].name;
            this.cities.push(string);
            break;
          }
        }
      }
      this.checkAll = this.cities.length === this.userList.length;
      this.isIndeterminate = this.cities.length > 0 && this.cities.length < this.userList.length;
    },
    // tree组件-节点过滤
    filterNode(value, data) {
      if (!value) return true;
      return data.label.indexOf(value) !== -1; // 注意：这里的label要与 propsTree里的配label字段相同
    },
    // 点击部门事件
    nodeClick(data) {
      this.orgObj.id = data.id;
      this.orgObj.label = data.label;
      this.filterPerson = undefined;
      this.getAllPage();
    },
    // 全选人员
    CheckAllPerson(boolean) {
      this.isIndeterminate = false;
      let arr = [];
      if(boolean){ // 全选
        for (let i = 0; i < this.userList.length; i++ ) {
          arr[i] = this.userList[i].id+"-"+this.userList[i].name;
          let index = this.personRoster.indexOf(arr[i]);
          if(index<0) {
            this.personRoster.push(arr[i]);
            this.cities = this.cities.concat(arr[i]);
          }
        }
      }else{ // 取消全选
        for (let i = 0; i < this.userList.length; i++ ) {
          arr[i] = this.userList[i].id+"-"+this.userList[i].name;
          let index = this.personRoster.indexOf(arr[i]);
          if(index>-1) this.personRoster.splice(index,1);
          this.cities.splice(index,1);
        }
      }
    },
    // 单选人员-返回的是勾选中数组
    RadioPerson(arr) {
      let checkedCount = arr.length;
      this.checkAll = checkedCount === this.userList.length;
      this.isIndeterminate = checkedCount > 0 && checkedCount < this.userList.length;
      this.addPerson(arr) // 判断这次单选了几个人员，重新处理最终选择的人员列表（personRoster）
    },
    // 单选后的人员
    addPerson(val){
      // 这里的目的是先把这个部门人员列表从最终选择的人员列表（personRoster）中全都删除掉，然后再把单选中的数组赋给personRoster
      for(let v of this.userList){
        let string = v.id+"-"+v.name;
        let index = this.personRoster.indexOf(string);
        if(index>-1) this.personRoster.splice(index,1);
      }
      // 把单选中的数组重新赋给personRoster,要确保赋值的对象obj在personRoster中不存在
      for(let obj of val){
        let index = this.personRoster.indexOf(obj);
        if(index<0) this.personRoster.push(obj);
      }
    },
    // 返回最终人员列表（personRoster）
    outcome(){
      let arr = [];
      for(let v of this.personRoster){ // 这里的v是id+'-'+name的字符串
        let obj = {id:v.split("-")[0],name:v.split("-")[1]}
        arr.push(obj)
      }
      this.$emit('outcome',arr);
    },
  }
}
</script>

<style lang="scss" scoped>
/deep/.dialogDiv{
  width:100%;
  box-sizing: border-box;
  .el-input{
    margin-bottom: 20px;
  }
  ul{
    width: 100%;
    padding: 0px;
    margin:0px;
    list-style-type: none;
    display: flex!important;
    flex-wrap: wrap;
    li{
      width: 50%!important;
      padding: 0px 10px;
    }
    li:nth-child(2){
      border-left:3px solid #999!important;
    }
  }
  .dialigGroup{
    max-height: 400px;
    overflow-y: auto;
  }
  .dialigGroup::-webkit-scrollbar {/*滚动条整体样式*/
    width: 10px;     /*高宽分别对应横竖滚动条的尺寸*/
    height: 1px;
  }
  .dialigGroup::-webkit-scrollbar-thumb {/*滚动条里面小方块*/
    border-radius: 10px;
    /*-webkit-box-shadow: inset 0 0 5px rgba(0,0,0,0.2);*/
    background: rgba(0,0,0,0.5);
  }
  .dialigGroup::-webkit-scrollbar-track {/*滚动条里面轨道*/
    /*-webkit-box-shadow: inset 0 0 5px rgba(0,0,0,0.2);*/
    border-radius: 10px;
    background: #fff;
  }
}
</style>
