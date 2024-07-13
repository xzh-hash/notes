绑定v-bind:id    简写 ：id
监听v-on:click 简写 @click
双向绑定 v-model
条件渲染 v-if  v-else v-else-if
列表渲染 v-for


计算属性 computed 

生命周期和模板引用
mounted(){
    this.$refs.pElementRef.textContent="1111"
  }

<p ref="pElementRef">hello</p>

侦听器
watch: {
    count(newCount) {
      // 没错，console.log() 是一个副作用
      console.log(`new count is: ${newCount}`)
    }
  }



组件
  import ChildComp from './ChildComp.vue'

export default {
  components: {
    ChildComp
  }
}