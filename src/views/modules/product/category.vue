<template>
  <div>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽"></el-switch>
    <el-button :disabled="!draggable" @click="batchUpdateDrop" type="primary" size="medium">保存拖拽</el-button>
    <el-button @click="batchDelete" type="danger" size="medium">批量删除</el-button>
    <el-tree :data="menus" :props="defaultProps" @node-click="handleNodeClick"
             :expand-on-click-node="false" show-checkbox node-key="catId"
             :default-expanded-keys="expandedKeys"
             :draggable="draggable"
             :allow-drop="allowDrop"
             @node-drop="handleDrop"
             ref="menuTree">
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button v-if="node.level <= 2" type="text" size="mini" @click="() => append(data)">添加</el-button>
          <el-button v-if="node.childNodes.length === 0" type="text" size="mini"
                     @click="() => remove(node, data)">删除</el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">编辑</el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog :title="dialogTitle" :visible.sync="dialogVisible" width="30%"
               :close-on-click-modal="false">
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
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
// 这里可以导入其他文件(比如：组件，工具 js，第三方插件 js，json文件，图片文件等等)
// 例如：import 《组件名称》 from '《组件路径》';

export default {
  // import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data () {
    return {
      // 是否开启拖拽, 默认是不可以拖拽的
      draggable: false,
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

    /**
     * trigger: 点击 “批量删除” 按钮
     * feature: 获取到选中的节点, 进行批量删除
     */
    batchDelete () {
      console.log('Trigger batchDelete()')

      // checkedNodes:选中的节点, 通过标签自带的方法getCheckedNodes()获取
      let checkedNodes = this.$refs.menuTree.getCheckedNodes()
      console.log('checkedNodes: ', checkedNodes)
      console.log('checkedNodes NameList: ', checkedNodes.map(x => x.name))

      // 将选中节点的ID组装成ID集合deletedCategoryIds
      let deletedCategoryIds = []
      for (let i = 0; i < checkedNodes.length; i++) {
        deletedCategoryIds.push(checkedNodes[i].catId)
      }

      // 1、给出操作提示
      this.$confirm(`是否批量删除【${checkedNodes.map(x => x.name)}】菜单？`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        // 2、调用后端删除逻辑
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(deletedCategoryIds, false)
        }).then(() => {
          // 2.1 删除成功, 前端给出提示
          this.$message({
            message: '菜单批量删除成功',
            center: true,
            type: 'success'
          })
          // 2.2 删除之后, 刷新菜单
          this.getMenus()
        })
      }).catch(() => {
        // 3、取消删除, 给出提示
        this.$message({
          message: '取消批量删除菜单',
          center: true,
          type: 'info'
        })
      })
    },

    /**
     * trigger: 点击 “保存拖拽” 按钮
     * feature: 批量提交拖拽后需要修改数据的节点数据给后端, 批量保存更新
     */
    batchUpdateDrop () {

    },

    /**
     * 拖拽成功后的后续逻辑回调处理, 拖拽成功后会触发该函数
     *
     * @param draggingNode x
     * @param dropNode x
     * @param dropType x
     * @param ev x
     */
    handleDrop (draggingNode, dropNode, dropType, ev) {
      console.log('handleDrop: ', draggingNode, dropNode, dropType)

      // 1、当前节点最新父节点的id
      let draggingNodeParentCId = 0
      let siblings = null

      if (dropType === 'before' || dropType === 'after') {
        draggingNodeParentCId = dropNode.parent.data.catId === undefined ? 0 : dropNode.parent.data.catId
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
      }).then(() => {
        this.$message({
          message: '菜单拖拽成功',
          type: 'success'
        })
      })
    },

    /**
     * 修改子节点的NodeLevel
     *
     * @param node
     */
    updateChildNodeLevel (node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          const cNode = node.childNodes[i].data
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level
          })
          this.updateChildNodeLevel(node.childNodes[i])
        }
      }
    },

    /**
     * 通过返回true/false判断来进行判断是否可以被拖拽
     *
     * @param draggingNode
     * @param dropNode
     * @param type
     * @returns {boolean}
     */
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

    /**
     * 抽象出计算节点总层数的方法, 入参是Node
     * @param node
     */
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

    /**
     * 获取菜单
     */
    getMenus () {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({data}) => {
        console.log('成功获取到菜单数据...', data.data)
        this.menus = data.data
      })
    },

    /**
     * append
     * @param data
     */
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

    /**
     * 删除
     *
     * @param node
     * @param data
     */
    remove (node, data) {
      console.log('remove button click', node, data)
      const ids = [data.catId]

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
          }).then(() => {
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

    /**
     * 修改
     *
     * @param data
     */
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

    /**
     * 策略, 添加还是修改
     */
    submitData () {
      if (this.dialogType === 'append') {
        this.addCategory()
      }
      if (this.dialogType === 'edit') {
        this.editCategory()
      }
    },

    /**
     * 点击确认, 向数据库执行添加三级分类
     */
    addCategory () {
      console.log('提交的三级分类数据：', this.category)

      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(() => {
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

    /**
     * 点击确认, 修改三级分类数据
     */
    editCategory () {
      const {catId, name, icon, productUnit} = this.category

      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData({catId, name, icon, productUnit}, false)
      }).then(() => {
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
