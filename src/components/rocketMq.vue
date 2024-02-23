<template>
  <div>
    <el-row>
      <el-descriptions class="margin-top" :column="1" border :labelStyle="{ width: '120px' }">
        <el-descriptions-item>
          <template slot="label">
            节点类型
          </template>
          RocketMQ
        </el-descriptions-item>
        <el-descriptions-item>
          <template slot="label">
            主题名
          </template>
          {{ currentNode.name }}
        </el-descriptions-item>
      </el-descriptions>
    </el-row>
  </div>
</template>

<script>
export default {
  name: 'rocketMq',
  components: {},
  props: {
    currentNode: {
      id: '',
      endpointType: '',
      name: ''
    }
  },
  watch: {
    currentNode(oldVal, newVal) {
    }
  },
  data() {
    return {}
  },
  created() {
  },
  mounted: function () {
  },

  methods: {}
}
</script>

<style>
</style>