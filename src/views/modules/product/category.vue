<!--  -->
<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <el-button
      type="danger"
      round
      @click="batchDelete"
    >批量删除</el-button>
    <el-tree
      :data="data"
      :props="defaultProps"
      @node-click="handleNodeClick"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span
        class="custom-tree-node"
        slot-scope="{ node, data }"
      >
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>

          <el-button
            type="text"
            size="mini"
            @click="() => edit(data)"
          >
            Edit
          </el-button>
          <el-button
            v-if="node.childNodes.length==0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input
            v-model="category.name"
            autocomplete="off"
          ></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input
            v-model="category.icon"
            autocomplete="off"
          ></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span
        slot="footer"
        class="dialog-footer"
      >
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button
          type="primary"
          @click="submitData"
        >确 定</el-button>
      </span>
    </el-dialog>
  </div>

</template>

<script>
//这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';

export default {
  //import引入的组件需要注入到对象中才能使用
  components: {},
  data() {
    return {
      draggable: false,
      updateNodes: [], //需要更新的节点的信息
      maxLevel: 0, // 最大深度
      title: "", //对话框的标题
      dialogType: "", // 对话框的类型：edit,add
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        icon: "",
        productUnit: "",
        catId: null,
      }, // 表单信息
      dialogVisible: false, //对话框的状态
      data: [], // 树的数据
      expandedKey: [], // 默认展开的节点的 key 的数组
      defaultProps: {
        children: "children", // 子树对应的属性值
        label: "name", //节点需要显示的内容
      },
    };
  },
  methods: {
    /**
     * 批量删除
     */
    batchDelete() {
      let catIds = this.$refs.menuTree.getCheckedKeys();
      console.log("catIds:", catIds);
      this.$confirm(`是否批量删除 所选的 菜单?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          if (catIds == null || catIds.length == 0) {
            this.$message({
              message: "所选菜单不能为空",
              type: "warning",
            });
            return;
          }
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(catIds, false),
          }).then(({ data }) => {
            this.$message({
              message: "菜单批量删除成功~",
              type: "success",
            });
            // 刷新页面
            this.getMenus();
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除",
          });
        });
    },
    /**
     * 当前拖拽成功后触发的事件
     * @param {*} draggingNode
     * @param {*} dropNode
     * @param {*} dropType
     * @param {*} ev
     */
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log("tree drop: ", dropNode.label, dropType);
      // 1.当前节点最新的父节点id
      let pCid = 0;
      let siblings = null;
      if (dropType == "before" || dropType == "after") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      }

      // 2.当前拖拽节点的最新顺序
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          // 如果遍历的正是当前正在拖拽的节点
          let catLevel = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            // 当前节点的层级发生变化
            catLevel = siblings[i].level;
            // 修改他子节点的层级
            this.updateChildNodeLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          //如果是相邻的节点
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }

      // 更新数据库中的数据
      console.log("updateNodes", this.updateNodes);
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单顺序修改成功~",
          type: "success",
        });
        // 刷新出新的菜单
        this.getMenus();
        // 设置需要默认展开的菜单
        this.expandedKey = [pCid];
        // 清空数据
        this.updateNodes = [];
        // this.maxLevel = 0;
      });
    },
    /**
     * 更新子节点的层级
     * @param {*} node
     */
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },
    /**
     * 当前拖拽是否允许，true：允许拖拽；false：不允许拖拽
     * @param {*} draggingNode  正在拖拽的节点
     * @param {*} dropNode      参考节点
     * @param {*} type          类型
     * deep算出的是当前节点拥有的深度。比如5楼到2楼有几楼高：5-2+1=4
     * 假设a挪到b中，则只要a拥有的深度+b的节点<=3就行
     * 假设a挪到b平级，则a+b的父节点<=3就行
     */
    allowDrop(draggingNode, dropNode, type) {
      // 被拖动的当前节点以及所在的父节点总层数不能大于3
      console.log("allowDrop:", draggingNode, dropNode, type);
      this.countNodeLevel(draggingNode.data); //计算子节点的maxLevel
      let deep = this.maxLevel - draggingNode.data.catLevel + 1;
      console.log("深度：", deep);
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    // 计算节点的maxLevel
    countNodeLevel(node) {
      this.maxLevel = node.catLevel;
      //找到所有子节点，求出最大深度
      if (node.children != null && node.children.length > 0) {
        for (let i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel) {
            this.maxLevel = node.children[i].catLevel;
          }
          this.countNodeLevel(node.children[i]);
        }
      }
    },
    handleNodeClick(data) {
      console.log(data);
    },
    // 获取分类菜单
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        this.data = data.data;
      });
    },
    // 当点击edit会触发
    edit(data) {
      console.log("要修改的数据:", data);
      this.title = "修改分类";
      this.dialogType = "edit";
      this.dialogVisible = true;
      // 发送请求获取当前节点的最新数据【这是为了防止在页面长时间停留，导致页面数据已经被其他人修改过了，所以不能直接用页面上的数据进行修改】
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        console.log("请求得到的数据,", data);
        this.category.name = data.data.name;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.catId = data.data.catId;
        this.category.parentCid = data.data.parentCid;
      });
    },
    // 当点击append会触发
    append(data) {
      this.title = "添加分类";
      this.dialogType = "add";
      this.category.name = ""; // 清空对话框
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1; //为了防止是字符串，所以需要*1
      this.category.catId = null;
      this.category.icon = "";
      this.category.productUnit = "";
      this.category.sort = 0;
      this.category.showStatus = 1;
    },

    //提交数据
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      } else if (this.dialogType == "edit") {
        this.editCategory();
      }
    },

    // 修改三级分类数据
    editCategory() {
      var { catId, name, icon, productUnit } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          type: "success",
          message: "菜单修改成功!",
        });
        // 设置需要默认展开的菜单[删除节点的父节点]
        this.expandedKey = [this.category.parentCid];
        // 刷新出新的菜单
        this.getMenus();
        // 关闭对话框
        this.dialogVisible = false;
      });
    },

    // 删除
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】菜单?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            this.$message({
              type: "success",
              message: "菜单删除成功!",
            });
            // 设置需要默认展开的菜单[删除节点的父节点]
            this.expandedKey = [node.parent.data.catId];
            // 刷新出新的菜单
            this.getMenus();
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除",
          });
        });
    },
    // 添加分类
    addCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          type: "success",
          message: "菜单保存成功!",
        });
        // 设置需要默认展开的菜单[删除节点的父节点]
        this.expandedKey = [this.category.parentCid];
        // 刷新出新的菜单
        this.getMenus();
        // 关闭对话框
        this.dialogVisible = false;
      });
    },
  },
  //监听属性 类似于data概念
  computed: {},
  //监控data中的数据变化
  watch: {},
  //方法集合
  //生命周期 - 创建完成（可以访问当前this实例）
  created() {
    this.getMenus();
  },
  //生命周期 - 挂载完成（可以访问DOM元素）
  mounted() {},
  beforeCreate() {}, //生命周期 - 创建之前
  beforeMount() {}, //生命周期 - 挂载之前
  beforeUpdate() {}, //生命周期 - 更新之前
  updated() {}, //生命周期 - 更新之后
  beforeDestroy() {}, //生命周期 - 销毁之前
  destroyed() {}, //生命周期 - 销毁完成
  activated() {}, //如果页面有keep-alive缓存功能，这个函数会触发
};
</script>
<style lang='scss' scoped>
//@import url(); 引入公共css类
</style>