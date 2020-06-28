<template>
  <div>
    <el-card>
      <CategorySelector @categoryChange="handleCategoryChange" ref="cs"></CategorySelector>
    </el-card>
    <el-card style="margin-top:20px;">

      <div v-show="isShowList">
        <el-button type="primary" icon="el-icon-plus" @click="showAdd" :disabled="!category3Id">添加属性</el-button>
        <el-table :data="attrs" border v-loading=loading >
          <el-table-column label="序号" type="index" width="80" align="canter"></el-table-column>
          <el-table-column label="属性名称" prop="attrName" width="150"></el-table-column>
          <el-table-column label="属性值列表">
            <template slot-scope="{row,$index}">
              <el-tag type="info" v-for="value in row.attrValueList" :key="value.id">
                {{value.valueName}}
              </el-tag>
            </template>
          </el-table-column>

          <el-table-column label="操作" width="150">
            <template slot-scope="{row,$index}">
              <hint-Button title="修改属性" type="primary" icon="el-icon-edit" size="mini"
              @click="showUpdate(row)"></hint-Button>
              <el-popconfirm :title="`确定删除${row.attrName}吗？`" @onConfirm="deleteAttr(row.id)">
                <HintButton slot="reference" title="删除属性" icon="el-icon-delete" size="mini" type="danger"></HintButton>
              </el-popconfirm>
            </template>
          </el-table-column>
        </el-table>
      </div>

       <div v-show="!isShowList">
         <el-form inline>
           <el-form-item label="属性名">
             <el-input type="text" placeholder="请输入属性名" v-model="attr.attrName"></el-input>
           </el-form-item>
         </el-form>

         <el-button type="primary" icon="el-icon-plus" @click="addAttrValue"
         :disabled="!attr.attrName">添加属性值</el-button>
         <el-button @click="isShowList=true">取消</el-button>

        <el-table border style="margin:20px 0" :data="attr.attrValueList">
          <el-table-column label="序号" type="index" width="80" align="canter"></el-table-column>
          <el-table-column label="属性值列表">
            <template slot-scope="{row,$index}">
              <el-input :ref="$index" v-if="row.edit" v-model="row.valueName" placeholder="请输入名称"
              @blur="toList(row)" @keyup.enter.native="toList(row)" size="mini"></el-input>
              <span v-else @click="toEdit(row,$index)" style="display:inline-block;width:100%">{{row.valueName}}</span>
            </template>
          </el-table-column>

          <el-table-column label="操作">
            <template slot-scope="{row,$index}">
              <el-popconfirm :title="`确定删除${row.valueName}吗？`" @onConfirm="attr.attrValueList.splice($index,1)">
                <HintButton slot="reference" title="删除属性" icon="el-icon-delete" size="mini" type="danger"></HintButton>
              </el-popconfirm>
            </template>
          </el-table-column>
        </el-table>

        <el-button type="primary" :disabled="!attr.attrName||attr.attrValueList.length===0" @click="addOrUpdate">保存</el-button>
        <el-button @click="isShowList=true">取消</el-button>
      </div>
    </el-card>
  </div>
</template>

<script>
import cloneDeep from 'lodash/cloneDeep'
export default {
  name: 'AttrList',
  data(){
    return{
      loading:false,//是否正在加载
      category1Id: null, // 一级分类ID
      category2Id: null, // 二级分类ID
      category3Id: null, // 三级分类ID
      attrs:[],//属性列表
      isShowList:true,//是否显示属性列表界面
      attr:{
        attrName:'',
        attrValueList:[],
        categoryId:'',
        categoryLevel:3
      }
    }
  },

  watch:{
    isShowList(value){
      this.$refs.cs.disabled=!value
    }
  },

  methods:{
    //删除属性
    async deleteAttr(id){
      const result = await this.$API.attr.remove(id)
      if(result.code===200){
        this.$message.success('删除属性成功')
        this.getAttrs()
      }else{
        this.$message.error('删除属性失败')
      }
    },

    //添加或更新属性
    async addOrUpdate(){
      const {attr} = this
      attr.attrValueList=attr.attrValueList.filter(attrValue=>{
        if(attrValue.valueName){
          delete attrValue.edit
          return true
        }
        return false
      })
      if(attr.attrValueList.length===0){
        this.$message.warning('至少指定一个属性值名称')
        return
      }

      const result = await this.$API.attr.save(attr)
      if(result.code===200){
        this.$message.success('保存属性成功')
        this.isShowList=true
        this.getAttrs()
      }else{
        this.$message.error('保存属性失败')
      }
    },

    //指定属性从编辑变为查看
    toList(attrValue){
      if(attrValue.valueName.trim()==='') return
      const isRepeat = this.attr.attrValueList.filter(value=>value.valueName===attrValue.valueName).length===2
      if(isRepeat){
        this.$message.warning('属性名不能重复')
        attrValue.valueName=''
        return
      }
      attrValue.edit=false
    },

    //从查看变为编辑
    toEdit(attrValue,index){
      if(attrValue.hasOwnProperty('edit')){
        attrValue.edit=true
      }else{
        this.$set(attrValue,'edit',true)
      }
      this.$nextTick(()=>this.$refs[index].focus())
    },

    //添加一个新的平台属性值
    addAttrValue(){
      const attrValue={
        attrId:this.id,
        valueName:'',
        edit:true,//标识2为编辑模式
      }
      this.attr.attrValueList.push(attrValue)
      this.$nextTick(()=>this.$refs[this.attr.attrValueList.length-1].focus())
    },

    //显示添加界面
    showAdd(){
      this.attr={
        attrName:'',
        attrValueList:[],
        categoryId:this.category3Id,
        categoryLevel:3
      }
      this.isShowList=false
    },

    //显示修改界面
    showUpdate(attr){
      this.attr=cloneDeep(attr)
      this.isShowList=false
    },

    //分类id发生改变的监听回调
    handleCategoryChange({categoryId,level}){
      if(level===1){
        this.category2Id=null
        this.category3Id=null
        this.attrs=[]
        this.category1Id=categoryId
      }else if(level===2){
        this.category3Id=null
        this.attrs=[]
        this.category2Id=categoryId
      }else if(level===3){
        this.category3Id=categoryId
        this.getAttrs()
      }
    },

    //异步获取属性列表
    async getAttrs(){
      const {category1Id,category2Id,category3Id}=this
      this.loading=true
      const result = await this.$API.attr.getList(category1Id, category2Id, category3Id)
      this.loading=false
      this.attrs=result.data
    }
  }
}
</script>

<style scoped>
</style>

