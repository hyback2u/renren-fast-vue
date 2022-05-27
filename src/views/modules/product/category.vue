<template>
  <div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      @node-click="handleNodeClick"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKeys"
      draggable
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >添加
          </el-button>
          <el-button
            v-if="node.childNodes.length === 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >删除
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            编辑
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="dialogTitle"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>

      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
// 这里可以导入其他文件（比如：组件，工具 js，第三方插件 js，json文件，图片文件等等）
// 例如：import 《组件名称》 from '《组件路径》';

export default {
  // import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data () {
    return {
      // 拖拽后需要修改的数据
      updateNodes: [],
      // maxLevel最大节点
      maxLevel: 0,
      // 这里，真正的数据要发送请求从后台数据库获取
      menus: [],
      // 默认要展开的节点
      expandedKeys: [],
      // 对话框默认关闭
      dialogVisible: false,
      // 对话框绑定的对象
      category: {
        name: '',
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        productUnit: '',
        icon: ''
      },
      // 区分是修改打开, 还是新增打开 edit/append
      dialogType: '',
      // 弹窗提示消息
      dialogTitle: '',
      defaultProps: {
        children: 'childCategoryEntity',
        label: 'name'
      }
    }
  },
  // 计算属性 类似于 data 概念
  computed: {},
  // 监控 data 中的数据变化
  watch: {},
  // 方法集合
  methods: {
    handleNodeClick (data) {
      console.log(data)
    },

    // 拖拽成功后的后续逻辑回调处理, 拖拽成功后会触发该函数
    handleDrop (draggingNode, dropNode, dropType, ev) {
      console.log('handleDrop: ', draggingNode, dropNode, dropType)

      // 1、当前节点最新父节点的id
      let draggingNodeParentCId = 0
      let siblings = null

      if (dropType === 'before' || dropType === 'after') {
        draggingNodeParentCId =
          dropNode.parent.data.catId === undefined
            ? 0
            : dropNode.parent.data.catId
        siblings = dropNode.parent.childNodes
      } else {
        // todo 为啥有时候取data, 有时候直接取不用data呀？
        // ===> node不取data, 走的是Node节点的数据结构, 否则, 走的是自定义Java模型的数据结构
        draggingNodeParentCId = dropNode.data.catId
        siblings = dropNode.childNodes
      }

      // 2、当前拖拽节点的最新顺序
      // if (dropType == "inner") {
      //   siblings = dropNode.childCategoryEntity;
      // } else {
      //   siblings = dropNode.parent.childCategoryEntity;
      // }
      // 排序
      for (let i = 0; i < siblings.length; i++) {
        // 拖拽还可能会改变父子关系id
        if (siblings[i].data.catId === draggingNode.data.catId) {
          // 如果遍历的是当前正在拖拽的节点, 不仅改变顺序, 还要改变父id --> 还有层级
          // draggingNodeDefaultCatLevel:当前节点默认层级
          let draggingNodeDefaultCatLevel = draggingNode.level
          if (siblings[i].level === draggingNode.level) {
            console.log('当前节点的层级发生变化')
            // 1、修改当前节点的层级
            draggingNodeDefaultCatLevel = siblings[i]
            // if (dropType == "before" || drop == "after") {
            //   draggingNodeDefaultCatLevel = dropNode.level;
            // } else {
            //   draggingNodeDefaultCatLevel = dropNode.level + 1;
            // }
            // 2、修改子节点的层级(递归修改) --> 写一个方法
            this.updateChildNodeLevel(siblings[i])
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: draggingNodeParentCId,
            catLevel: draggingNodeDefaultCatLevel
          })
        } else {
          this.updateNodes.push({catId: siblings[i].data.catId, sort: i})
        }
      }

      // 3、当前拖拽节点的最新层级
      console.log('updateNodes', this.updateNodes)

      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({data}) => {
        this.$message({
          message: '菜单拖拽成功',
          type: 'success'
        })
      })
    },

    // 修改子节点的NodeLevel
    updateChildNodeLevel (node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level
          })
          this.updateChildNodeLevel(node.childNodes[i])
        }
      }
    },

    // 通过返回true/false判断来进行判断是否可以被拖拽
    allowDrop (draggingNode, dropNode, type) {
      // 1、被拖动的当前节点以及所在的父节点，总层数不能大于3

      // 2、被拖动的当前节点(draggingNode)总层数
      console.log('allowDrop: ', draggingNode, dropNode, type)

      // 抽象出计算节点总层数的方法countNodeLevel()
      this.countNodeLevel(draggingNode)

      // 当前正在拖动的节点 + 父节点所在的深度不大于3即可
      console.log('深度：', this.maxLevel)
      let deep = this.maxLevel - draggingNode.data.catLevel + 1
      console.log('deep', deep)

      if (type === 'inner') {
        // 注意这里是节点的level, 并不是节点data里的catLevel
        return deep + dropNode.level <= 3
      } else {
        return deep + dropNode.parent.level <= 3
      }
    },

    // 抽象出计算节点总层数的方法, 入参是Node
    countNodeLevel (node) {
      // 说明有子节点
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level
          }
          this.countNodeLevel(node.childNodes[i])
        }
      }
    },

    getMenus () {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({data}) => {
        console.log('成功获取到菜单数据...', data.data)
        this.menus = data.data
      })
    },
    append (data) {
      console.log('append button click', data)

      this.dialogType = 'append'
      this.dialogTitle = '添加分类'
      this.dialogVisible = true // 打开对话框

      this.category.parentCid = data.catId // 父节点Id, 就是在哪个节点进行点击的Append, 那就是哪个
      this.category.catLevel = data.catLevel * 1 + 1 // 层级, 就是当前层级 + 1

      this.category.catId = null
      this.category.name = ''
      this.category.icon = ''
      this.category.productUnit = ''
      this.category.sort = 0
      this.category.showStatus = 1
    },

    remove (node, data) {
      console.log('remove button click', node, data)
      var ids = [data.catId]

      this.$confirm(`是否删除【${data.name}】菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(ids, false)
          }).then(({data}) => {
            console.log('删除成功')
            // 删除成功后, 重新请求所有的菜单并展示
            this.$message({
              message: '菜单删除成功',
              type: 'success'
            })
            // 刷新出新的菜单
            this.getMenus()
            // 设置需要默认展开的菜单
            this.expandedKeys = [node.parent.data.catId]
          })
        })
        .catch(() => {
          this.$message({
            type: 'info',
            message: '已取消删除'
          })
        })
    },

    // 修改
    edit (data) {
      console.log('要修改的数据：', data)
      this.dialogTitle = '修改分类'

      this.dialogType = 'edit'
      this.dialogVisible = true

      // 这里回显, 不能直接取, 如果没刷新可能就是旧的, 应该发送请求获取当前节点最新的数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get'
      }).then(({data}) => {
        // 请求成功
        console.log('要回显的数据：', data)
        this.category.name = data.data.name
        this.category.icon = data.data.icon
        this.category.productUnit = data.data.productUnit
        this.category.parentCid = data.data.parentCid
      })
      this.category.name = data.name

      this.category.catId = data.catId
    },

    // 策略, 添加还是修改
    submitData () {
      if (this.dialogType === 'append') {
        this.addCategory()
      }
      if (this.dialogType === 'edit') {
        this.editCategory()
      }
    },

    // 点击确认, 向数据库执行添加三级分类
    addCategory () {
      console.log('提交的三级分类数据：', this.category)

      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({data}) => {
        console.log('添加分类成功')
        // 添加分类成功, 重新请求所有的菜单并展示
        this.$message({
          message: '添加分类成功',
          type: 'success'
        })
        // 关闭对话框
        this.dialogVisible = false

        // 刷新出新的菜单
        this.getMenus()
        // 设置需要默认展开的菜单
        this.expandedKeys = [this.category.parentCid]
      })
    },

    // 点击确认, 修改三级分类数据
    editCategory () {
      var {catId, name, icon, productUnit} = this.category

      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData({catId, name, icon, productUnit}, false)
      }).then(({data}) => {
        console.log('分类修改成功')
        // 分类修改成功, 重新请求所有的菜单并展示
        this.$message({
          message: '分类修改成功',
          type: 'success'
        })
        // 关闭对话框
        this.dialogVisible = false

        // 刷新出新的菜单
        this.getMenus()
        // 设置需要默认展开的菜单
        this.expandedKeys = [this.category.parentCid]
      })
    }
  },
  // 生命周期 - 创建完成（可以访问当前 this 实例）
  created () {
    this.getMenus()
  },
  // 生命周期 - 挂载完成（可以访问 DOM 元素）
  mounted () {
  },
  beforeCreate () {
  }, // 生命周期 - 创建之前
  beforeMount () {
  }, // 生命周期 - 挂载之前
  beforeUpdate () {
  }, // 生命周期 - 更新之前
  updated () {
  }, // 生命周期 - 更新之后
  beforeDestroy () {
  }, // 生命周期 - 销毁之前
  destroyed () {
  }, // 生命周期 - 销毁完成
  activated () {
  } // 如果页面有 keep-alive 缓存功能，这个函数会触发
}
</script>
<style lang='scss' scoped>
// @import url(); 引入公共 css 类
</style>
