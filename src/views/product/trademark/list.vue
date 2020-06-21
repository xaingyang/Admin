<template>
  <div>
    <el-button type="primary" icon="el-icon-plus" @click="showAdd"
      >添加</el-button
    >
    <el-table style="margin:20px 0" :data="trademarks" border>
      <el-table-column label="序号" type="index" width="80" align="center">
      </el-table-column>

      <el-table-column prop="tmName" label="品牌名称" />
      <el-table-column label="品牌LOGO">
        <template slot-scope="scope">
          <img :src="scope.row.logoUrl" style="width:100px;height:50px;" />
        </template>
      </el-table-column>

      <el-table-column label="操作">
        <template slot-scope="{ row, $index }">
          <el-button
            type="warning"
            size="mini"
            icon="el-icon-edit"
            @click="showUpdate(row)"
            >修改</el-button
          >
          <el-button
            type="danger"
            size="mini"
            icon="el-icon-delete"
            @click="remove(row)"
            >删除</el-button
          >
        </template>
      </el-table-column>
    </el-table>

    <el-pagination
      style="text-align:center"
      :current-page="page"
      :page-sizes="[3, 6, 9]"
      :page-size="limit"
      layout=" prev, pager, next,jumper, ->,sizes ,total"
      :total="total"
      @current-change="getTrademarks"
      @size-change="handleSizeChange"
    />

    <el-dialog
      :title="form.id ? '修改品牌' : '添加品牌'"
      :visible.sync="isShowDialog"
    >
      <el-form :model="form" :rules="rules" ref="ruleForm" style="width:80%">
        <el-form-item label="品牌名称" label-width="100px" prop="tmName">
          <el-input v-model="form.tmName" autocomplete="off"></el-input>
        </el-form-item>

        <el-form-item label="品牌LOGO" label-width="100px" prop="logoUrl">
          <!-- action处理上传图片请求的地址 -->

          <el-upload
            class="avatar-uploader"
            action="/dev-api/admin/product/fileUpload"
            :show-file-list="false"
            :on-success="handleAvatarSuccess"
            :before-upload="beforeAvatarUpload"
          >
            <img v-if="form.logoUrl" :src="form.logoUrl" class="avatar" />
            <i v-else class="el-icon-plus avatar-uploader-icon"></i>
            <div slot="tip" class="el-upload__tip">
              只能上传jpg/png文件，且不超过50k
            </div>
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
  name: "Trademark",
  data() {
    return {
      trademarks: [],
      total: 0, //总数量
      page: 1, //当前页码
      limit: 3, //每页数量

      isShowDialog: false,
      form: {
        tmName: "",
        logoUrl: ""
      },
      rules: {
        tmName: [
          {
            required: true,
            message: "请输入品牌名称"
          },
          //内置校验规则
          // {
          //   min: 2,
          //   max: 10,
          //   message: "长度在 2 到 10 个字符",
          //   trigger: "blur"
          // }
          //自定义校验规则
          {
            validator: this.validateTmName,
            trigger: "blur"
          }
        ],
        logoUrl: [
          {
            required: true,
            message: "LOGO必须指定",
            trigger: "change"
          }
        ]
      }
    };
  },
  mounted() {
    // 异步获取第一页列表显示
    this.getTrademarks(); // 如果不用this报错: getTrademarks is not defined
  },

  methods: {
    //自定义校验品牌名称
    validateTmName(rule, value, callback) {
      if (value.length < 2 || value.length > 10) {
        callback("长度必须在2~10之间");
      } else {
        callback();
      }
    },
    remove(trademark) {
      this.$confirm(`确定删除 ${trademark.tmName} 吗?`, "提示", {
        type: "warning"
      })
        .then(async () => {
          // 点击确定的回调
          // 发删除品牌的请求
          const result = await this.$API.trademark.remove(trademark.id);
          // 如果成功了, 提示成功, 重新获取列表(哪一页?)
          if (result.code === 200) {
            this.$message({
              type: "success",
              message: "删除成功!"
            });

            // 哪一页?  显示上一页(当前页的列表数据只剩下1个)  否则显示当前页
            // 如果当前是第1页且只剩下1条数据 ==> 请求第1页数据(当前页)
            this.getTrademarks(
              this.trademarks.length === 1 && this.page > 1
                ? this.page - 1
                : this.page
            );
          } else {
            // 如果失败了, 提示删除失败
            this.$message({
              type: "error",
              message: "删除失败"
            });
          }
        })
        .catch(() => {
          // 点击取消的回调
          this.$message({
            type: "info",
            message: "已取消删除"
          });
        });
    },
    addOrUpdate() {
      this.$refs["ruleForm"].validate(async valid => {
        if (valid) {
          const trademark = this.form;
          let result;
          if (trademark.id) {
            result = await this.$API.trademark.update(trademark);
          } else {
            result = await this.$API.trademark.add(trademark);
          }
          if (result.code === 200) {
            this.$message.success(`${trademark.id ? "更新" : "添加"}成功！`);
            this.isShowDialog = false;
            this.getTrademarks(trademark.id ? this.page : 1);
          } else {
            this.$message.success(`${trademark.id ? "更新" : "添加"}成功！`);
          }
        } else {
          console.log("校验不通过！");
          return false;
        }
      });
    },
    handleAvatarSuccess(res, file) {
      this.form.logoUrl = res.data;
    },
    beforeAvatarUpload(file) {
      const isJPGOrPNG =
        file.type === "image/jpeg" || file.type === "image/png";
      const isLt50k = file.size / 1024 < 50;

      if (!isJPGOrPNG) {
        this.$message.error("上传头像图片只能是 JPG/PNG 格式!");
      }
      if (!isLt50k) {
        this.$message.error("上传头像图片大小不能超过 50K!");
      }
      return isJPGOrPNG && isLt50k;
    },
    showAdd() {
      this.form = {
        tmName: "",
        logoUrl: ""
      };
      this.isShowDialog = true;
    },
    showUpdate(trademark) {
      // this.form = trademark;
      this.form = {
        ...trademark
      };
      this.isShowDialog = true;
    },

    handleSizeChange(pageSize) {
      this.limit = pageSize;
      this.getTrademarks(1);
    },

    async getTrademarks(page = 1) {
      this.page = page;
      const result = await this.$API.trademark.getList(page, this.limit);
      if (result.code === 200) {
        const { records, total } = result.data;
        this.trademarks = records;
        this.total = total;
      } else {
        this.$message({
          type: "warning",
          message: "获取分页列表失败"
        });
      }
    }
  }
};
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
  border-color: #409eff;
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
