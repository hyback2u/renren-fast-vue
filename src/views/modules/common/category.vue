<template>
  <el-tree :data="menus" :props="defaultProps" node-key="catId" ref="menuTree"
           @node-click="nodeClick"></el-tree>
</template>

<script>
export default {
  // import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data () {
    return {
      // 这里，真正的数据要发送请求从后台数据库获取
      menus: [],
      defaultProps: {
        children: 'children',
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
    /**
     * trigger: tree组件事件触发(节点被点击时的回调)后回调
     * feature: tree组件自带回调方法
     *
     * @param data 传递给data属性的数组中该节点所对应的对象
     * @param node 节点对应的 Node
     * @param component 节点组件本身
     */
    nodeClick (data, node, component) {
      console.log('--------> Trigger nodeClick()', data, node, component)
      console.log('子组件category的节点被点击')

      // 向父组件发送事件, 并携带上这3个数据
      this.$emit('tree-node-click', data, node, component)
    },

    getMenus () {
      console.log('Trigger getMenus()')
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({data}) => {
        console.log('成功获取到菜单数据...', data.data)
        this.menus = data.data
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
</style>
