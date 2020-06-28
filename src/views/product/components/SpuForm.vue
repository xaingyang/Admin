<template>
  <el-form label-width="100px" v-show="visible">
    <el-form-item label="SPU名称">
      <el-input placeholder="SPU名称" v-model="spuInfo.spuName"></el-input>
    </el-form-item>

    <el-form-item label="品牌">
      <el-select v-model="spuInfo.tmId" placeholder="请选择品牌">
        <el-option  :label="tm.tmName" :value="tm.id" v-for="tm in trademarkList" :key="tm.id"></el-option>
      </el-select>
    </el-form-item>

    <el-form-item label="SPU描述">
      <el-input type="textarea" placeholder="SPU描述" rows="4" v-model="spuInfo.description"></el-input>
    </el-form-item>

    <el-form-item label="SPU图片">
      <el-upload
        action="/dev-api/admin/product/fileUpload"
        list-type="picture-card"
        :on-preview="handlePictureCardPreview"
        :on-remove="handleRemove"
        :on-success="handleUploadSuccess"
        :file-list="spuImageList">
        <i class="el-icon-plus"></i>
      </el-upload>
      <el-dialog :visible.sync="dialogVisible">
        <img width="100%" :src="dialogImageUrl" alt="">
      </el-dialog>
    </el-form-item>
    <el-form-item label="销售属性">
      <el-select v-model="attrIdAttrName" :placeholder="unUsedSaleAttrList.length===0?'没有啦':`还有${unUsedSaleAttrList.length}个未选择`">
        <el-option :label="attr.name" v-for="attr in unUsedSaleAttrList" :key="attr.id" :value="attr.id+':'+attr.name"></el-option>
      </el-select>

      <el-button type="primary" icon="el-icon-plus" :disabled="!attrIdAttrName" @click="addSpuSaleAttr">添加销售属性</el-button>

      <el-table border style="margin-top: 20px" :data="spuInfo.spuSaleAttrList">
        <el-table-column label="序号" type="index" width="80" align="center"></el-table-column>
        <el-table-column label="属性名" prop="saleAttrName" width="150"></el-table-column>
        <el-table-column label="属性值名称列表">
          <template slot-scope="{row, $index}">
            <el-tag
              :key="attrValue.id"
              v-for="(attrValue, index) in row.spuSaleAttrValueList"
              closable
              :disable-transitions="false"
              @close="row.spuSaleAttrValueList.splice(index, 1)"
              >
              {{attrValue.saleAttrValueName}}
            </el-tag>
            <el-input
              class="input-new-tag"
              v-if="row.edit"
              :ref="$index"
              v-model.trim="row.saleAttrValueName"
              size="small"
              placeholder="名称"
              @keyup.enter.native="addSpuSaleAttrValue(row, $index)"
              @blur="addSpuSaleAttrValue(row, $index)"
            >
            </el-input>
            <el-button v-else class="button-new-tag" size="small"
              @click="showInput(row, $index)">+ 添加</el-button>
          </template>
        </el-table-column>
        <el-table-column label="操作" width="150" >
          <template slot-scope="{row,$index}">
            <hint-button title="删除" type="danger" icon="el-icon-delete" size="mini"
          @click="spuInfo.spuSaleAttrList.splice(index, 1)" />
          </template>
        </el-table-column>
      </el-table>
    </el-form-item>

    <el-form-item>
      <el-button type="primary" @click="save">保存</el-button>
      <el-button @click="back">取消</el-button>
    </el-form-item>

  </el-form>
</template>

<script>
export default {
  name: 'SpuForm',

  props: {
    visible: Boolean
  },

  data () {
    return {
      dialogImageUrl: '',
      dialogVisible: false,
      spuInfo:{
        category3Id:'',
        spuName:'',
        description:'',
        tmId:'',
        spuImageList:[],
        spuSaleAttrList:[],
      },
      spuImageList:[],
      trademarkList:[],
      saleAttrList:[],
      attrIdAttrName:'',
      spuId:''
    }
  },

  methods: {
    //重置数据
    resetData(){
       this.dialogImageUrl = '' // 大图的url
      this.dialogVisible = false // 标识大图dilaog是否显示

      this.spuId = '' // SPU ID
      this.spuInfo =  { // SPU详情信息对象
        category3Id: null, // 3级分类ID
        spuName: '', // spu名称
        description: '', // spu描述
        tmId: null, // spu的品牌id
        spuSaleAttrList: [], // spu的销售属性列表
        spuImageList: [], // spu图片列表
      }
      this.spuImageList = [] // SPU图片列表
      this.trademarkList = [] // 品牌列表
      this.saleAttrList = [] // 销售属性列表
      this.attrIdAttrName = '' // 用来收集销售属性id与name   id:name
    },

    //保存数据
    async save(){
      const {spuInfo,spuImageList} =this
      spuInfo.spuImageList=spuImageList.map(item=>({
        imaName:item.name,
        imgUrl:item.response?item.response.data:item.url
      }))
      spuInfo.spuSaleAttrList=spuInfo.spuSaleAttrList.filter(attr=>{
        delete attr.edit
        delete attr.saleAttrValueName
        return attr.spuSaleAttrValueList.length>0
      })
      //删除不必要的属性数据


      const result = await this.$API.spu.addUpdate(spuInfo)
      if(result.code===200){
        this.$message.success('保存SPU成功')
        this.$emit('updata:visible',false)
        this.$emit('success')
        this.resetData()
      }else{
        this.$message.error('保存SPU失败')
      }
    },

    //添加新的销售属性值对象
    addSpuSaleAttrValue(spuSaleAttr,index){
      const {baseSaleAttrId,saleAttrValueName} = spuSaleAttr
      if(!saleAttrValueName){
        spuSaleAttr.edit=false
        return
      }
      const isRepeat =  spuSaleAttr.spuSaleAttrValueList.some(value=>value.saleAttrValueName==saleAttrValueName)
      if(isRepeat){
        this.$message.warning('不能重复')
        this.$nextTick(()=>this.$refs[index].focus())
        return
      }
      spuSaleAttr.spuSaleAttrValueList.push({
        saleAttrValueName,
        baseSaleAttrId
      })
      spuSaleAttr.edit = false
      spuSaleAttr.saleAttrValueName=''
    },

    //显示当前行的输入框
    showInput(spuSaleAttr,index){
      if(spuSaleAttr.hasOwnProperty('edit')){
        spuSaleAttr.edit = true
      }else{
        this.$set(spuSaleAttr,'edit',true)
      }
      this.$nextTick(()=>this.$refs[index].focus())
    },

    //添加一个spu销售属性对象
    addSpuSaleAttr(){
      const [baseSaleAttrId,saleAttrName]=this.attrIdAttrName.split(':')
      const spuSaleAttr={
        baseSaleAttrId,
        saleAttrName,
        spuSaleAttrValueList:[],
      }
      this.spuInfo.spuSaleAttrList.push(spuSaleAttr)
      this.attrIdAttrName=''
    },

    //请求加载更新界面初始显示
    initLoadUpdateData(spuId){
      this.spuId=spuId
      this.getSpuInfo()
      this.getSpuImageList()
      this.getTrademarkList()
      this.getSaleAttrList()
    },

    //添加界面初始显示数据
    initLoadAddData(category3Id){
      this.spuInfo.category3Id=category3Id
      this.getTrademarkList()
      this.getSaleAttrList()
    },

    //获取spuInfo
    async getSpuInfo(){
      const result = await this.$API.spu.get(this.spuId)
      const spuInfo = result.data
      this.spuInfo=spuInfo
    },

    //获取spuImageList
    async getSpuImageList(){
      const result = await this.$API.sku.getSpuImageList(this.spuId)
      const spuImageList = result.data
      spuImageList.forEach(item=>{
        item.name=item.imgName
        item.url=item.imgUrl
      })
      this.spuImageList=spuImageList
    },

    //获取trademarkList
    async getTrademarkList(){
      const result = await this.$API.trademark.getList()
      const trademarkList = result.data
      this.trademarkList=trademarkList
    },

    //获取saleAttrList
    async getSaleAttrList(){
      const result = await this.$API.spu.getSaleAttrList()
      const saleAttrList = result.data
      this.saleAttrList=saleAttrList
    },


    /*
    返回
    */
    back () {
      // this.$parent.$parent.isShowSpuForm = false
      // 分发.sync内部绑定的update:visible的自定义事件
      this.$emit('update:visible', false)
      this.$emit('cancel')
      this.resetData()
    },


    //点击删除图片的回调函数
    handleRemove(file, fileList) {
      // console.log(file, fileList);
      this.spuImageList=fileList
    },

    //显示指定的大图
    handlePictureCardPreview(file) {
      this.dialogImageUrl = file.url;
      this.dialogVisible = true;
    },

    //上传图片成功后的回调
    handleUploadSuccess(response,file,fileList){
      this.spuImageList=fileList
    }
  },

  computed:{
    //未使用的销售属性数组
    unUsedSaleAttrList(){
      const {saleAttrList} = this
      const {spuSaleAttrList} = this.spuInfo
      const arr = saleAttrList.filter(attr=>{
        const spuAttr=spuSaleAttrList.find(spuAttr=>spuAttr.saleAttrName===attr.name)
        return !spuAttr
      })
      return arr
    }
  }
}
</script>

