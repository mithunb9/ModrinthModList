<template>
  <el-table
    :header-cell-style="{ color: '#F7FAFF', background: '#14213D' }"
    :default-sort="{ prop: 'name', order: 'ascending' }"
    sortable
    height="750"
    border
    :data="data"
    style="width: 100%"
  >
    <el-table-column prop="icon_url" label="Icon" width="80" fixed>
      <template v-slot="scope">
        <el-avatar shape="square" width="50" :src="scope.row.icon_url" />
      </template>
    </el-table-column>

    <el-table-column label="Name" width="180" fixed>
      <template v-slot="scope">
        <a target="_blank" :href="scope.row.modUrl">{{ scope.row.name }}</a>
      </template>
    </el-table-column>

    <el-table-column
      v-for="col in columns"
      :key="col"
      :prop="col"
      :label="formatTerm(col)"
    ></el-table-column>

    <el-table-column prop="fileUrl" label="Download" width="180" fixed="right">
      <template v-slot="scope">
        <a :href="scope.row.fileUrl"> {{ scope.row.filename }} </a>
      </template>
    </el-table-column>

    <el-table-column label="Delete" fixed="right">
      <template v-slot="scope">
        <el-button size="mini" type="danger" @click="handleDelete(scope.$index)"
          >Delete</el-button
        >
      </template>
    </el-table-column>
  </el-table>
</template>

<script>
export default {
  name: "ModView",
  props: {
    data: Object,
    columns: [],
    exportData: Array,
  },
  methods: {
    handleDelete(index) {
      console.log(this.$props.exportData);
      this.$props.exportData.forEach((item) => {
        if (item == this.$props.data[index].id) {
          this.$props.exportData.splice(index, 1);
        }
      });
      this.$props.data.splice(index, 1);
    },
    formatTerm(word) {
      if (Array.isArray(word)) {
        word.forEach((item, index) => {
          word[index] = item[0].toUpperCase() + item.slice(1);
        });

        return word.join(", ");
      } else {
        return (word[0].toUpperCase() + word.slice(1)).replace("_", " ");
      }
    },
  },
};
</script>
