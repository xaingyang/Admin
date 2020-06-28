<template>
  <div>
    <el-button type="primary" icon="el-icon-plus" @click="showAdd">添加</el-button>

    <el-table v-loading="loading" style="margin: 20px 0" :data="trademarks" border>
      <el-table-column label="序号" type="index" width="80" align="center" />

      <el-table-column prop="tmName" label="品牌名称" />

      <el-table-column label="品牌LOGO">
        <template slot-scope="scope">
          <img :src="scope.row.logoUrl" style="width: 100px; height:60px;" />
        </template>
      </el-table-column>

      <el-table-column label="操作">
        <template slot-scope="{row, $index}">
          <el-button type="warning" size="mini" icon="el-icon-edit" @click="showUpdate(row)">修改</el-button>
          <el-button type="danger" size="mini" icon="el-icon-delete" @click="remove(row)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>

    <el-pagination style="text-align: center" :current-page="page" :page-sizes="[3, 6, 9]" :page-size="limit"
      :total="total" layout="prev, pager, next, jumper, ->, sizes, total" @current-change="getTrademarks"
      @size-change="handleSizeChange" />

    <el-dialog :title="form.id ? '修改品牌' : '添加品牌'" :visible.sync="isShowDialog">
      <el-form :model="form" :rules="rules" ref="ruleForm" style="width: 80%">
        <el-form-item label="品牌名称" label-width="100px" prop="tmName">
          <el-input v-model="form.tmName" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="品牌LOGO" label-width="100px" prop="logoUrl">
          <el-upload class="avatar-uploader" action="/dev-api/admin/product/fileUpload" :show-file-list="false"
            :on-success="handleAvatarSuccess" :before-upload="beforeAvatarUpload">
            <img v-if="form.logoUrl" :src="form.logoUrl" class="avatar">
            <i v-else class="el-icon-plus avatar-uploader-icon"></i>
            <div class="el-upload__tip" slot="tip">只能上传jpg/png文件，且不超过100kb</div>
          </el-upload>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="isShowDialog = false">取 消</el-button>
        <el-button type="primary" @click="addOrUpdate">确 定</el-button>
      </div>
    </el-dialog>

  </div>
</template>

<script>
  export default {
    name: 'Trademark',

    data(){
      return{
        trademarks:[],
        total:0,
        page:1,
        limit:3,
        isShowDialog:false,
        form:{
          tmName:'',
          logoUrl:''
        },
        loading:false,
        rules:{
          tmName:[{
            required:true,
            message:'请输入名称',
          },
          {
            validator:this.validateTmName,
            trigger:'blur'
          }],
          logoUrl:[{
            required:true,
            message:'logo必须指定'
          }]
        }
      }
    },

    mounted(){
      this.getTrademarks()
    },

    methods:{
      //异步获取指定页码的列表数据显示
      async getTrademarks(page=1){
        this.page=page
        this.loading=true
        const result = await this.$API.trademark.getList(page,this.limit)
        this.loading=false
        if(result.code===200){
          const{
            records,
            total
          } = result.data
          this.trademarks=records
          this.total=total
        }else{
          this.$message.error('获取分页列表失败')
        }
      },

      //长度必须在2-10个之间
      validateTmName(rule,value,callback){
        if(value.length<2||value.length>10){
          callback('长度必须在2-10个之间')
        }else{
          callback()
        }
      },

      //删除指定的品牌
      remove(trademark){
        this.$confirm(`确定删除${trademark.tmName}吗？`,'提示',{
          type:'warning'
        }).then(async ()=>{
          const result = await this.$API.trademark.remove(trademark.id)
          if(result.code===200){
            this.$message.success('删除成功')
            this.getTrademarks(this.trademarks.length===1&&this.page>1?this.page-1:this.page)
          }else{
            this.$message.error('删除失败')
          }
        }).catch(()=>{
          this.$message.info('已取消删除')
        })
      },

      //添加或更新
      addOrUpdate(){
          this.$refs['ruleForm'].validate(async (valid) => { // 校验完成的回调
          if(valid){
            const trademark=this.form
            let result
            if(trademark.id){
              result = await this.$API.trademark.update(trademark)
            }else{
              result = await this.$API.trademark.add(trademark)
            }
            if(result.code===200){
              this.$message.success(`${trademark.id?'更新':'添加'}成功`)
              this.isShowDialog=false
              this.getTrademarks(trademark.id?this.page:1)
            }else{
              this.$message.error(`${trademark.id?'更新':'添加'}失败`)
            }
          }
        })
      },

      //上传成功时的回调
      handleAvatarSuccess(res,file){
        this.form.logoUrl=res.data
      },

      //检查文件大小
      beforeAvatarUpload(file){
        const isJPGOrPNG=['image/jpeg','image/png'].indexOf(file.type)>=0
        const isLt50k=file.size/1024<100
        if(!isJPGOrPNG){
          this.$message.error('上传头像图片只能是 JPG/PNG 格式!')
        }
        if(!isLt50k){
          this.$message.error('上传头像图片大小不能超过 100K!')
        }
        return isJPGOrPNG&&isLt50k
      },

      //显示修改界面
      showUpdata(trademark){
        this.form={
          ...trademark
        }
        this.isShowDialog=true
      },

      //显示添加的界面
      showAdd(){
        this.form={tmName:'',logoUrl:''}
        this.isShowDialog=true
        this.$nextTick(()=>{
          this.$refs.ruleForm.clearValidate()
        })
      },

      //修改每页数量的监听
      handleSizeChange(pageSize){
        this.limit=pageSize
        this.getTrademarks(1)
      },
    }
  }
</script>

<style>
 .avatar-uploader .el-upload {
    border: 1px dashed #d9d9d9;
    border-radius: 6px;
    cursor: pointer;
    position: relative;
    overflow: hidden;
  }

  .avatar-uploader .el-upload:hover {
    border-color: #409EFF;
  }

  .avatar-uploader-icon {
    font-size: 28px;
    color: #8c939d;
    width: 178px;
    height: 178px;
    line-height: 178px;
    text-align: center;
  }

  .avatar {
    width: 178px;
    height: 178px;
    display: block;
  }

</style>
