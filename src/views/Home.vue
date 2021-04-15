<template>
	<div>
		<div class="flexContainer">
			<el-input type="text" class="searchBox" v-model="query" />
			<el-button class="submitBtn" width="10rem"  icon="el-icon-search" @click="search()" :loading="loadingStatus" round size="medium">Search</el-button>
		</div>
		<div class="flexContainer">
			<el-input type="text" class="searchBox" v-model="importQuery" />
			<el-button class="submitBtn" width="10rem" background="#fca311" icon="el-icon-upload" @click="importModList()" :loading="loadingStatus" round size="medium">Import</el-button>
		</div>  

    <div id="selectBox" class="flexContainer">
      <el-checkbox :indeterminate="isIndeterminate" v-model="checkAll" @change="handleSelectAllChange">Check all</el-checkbox>
      <el-checkbox-group size="big" v-model="columns">
        <el-checkbox v-for="col in columnOptions" :label="col" :key="col">{{formatTerm(col)}}</el-checkbox>
      </el-checkbox-group>
    </div>
    <el-table :header-cell-style="{ color: '#F7FAFF', background: '#14213D' }" :default-sort = "{prop: 'name', order: 'ascending'}" height="750" border :data="displayedData" style="width: 100%"> 

      <el-table-column prop="icon_url" label="Icon" width="80" fixed>
        <template v-slot="scope">
        <el-avatar shape="square" width="50" :src="scope.row.icon_url"/>
        </template>
      </el-table-column>
      
      <el-table-column prop="name" label="Name" width="180" fixed></el-table-column>

      <el-table-column v-for="col in columns" :key="col" :prop="col" :label="formatTerm(col)"></el-table-column>
      
      <el-table-column prop="fileUrl" label="Download" width="180" fixed="right">
        <template v-slot="scope">
          <a :href="scope.row.fileUrl"> {{ scope.row.filename }} </a> 
        </template> 
      </el-table-column>

      <el-table-column label="Delete" fixed="right">  
        <template v-slot="scope">
          <el-button
            size="mini"
            type="danger"
            @click="handleDelete(scope.$index)">Delete</el-button>
        </template>
      </el-table-column>

    </el-table>
		<el-input type="textarea" v-model="exportData"></el-input>
	</div>
</template>

<script>
import axios from "axios";
import Url from "url-parse";
import { ref } from "vue";

const defaultOptions = ['followers','desc','published','updated','client','server','categories','license','game_versions','version_type','loader'];
export default {
  name: 'Home',
  components: {},
  data() {
    return {
      query: ref(''),
      importQuery: ref(''),
      exportData: ref([]),
      displayedData: [],
      columnOptions: defaultOptions,
      columns: defaultOptions,
      isIndeterminate: true,
      checkAll: false,
      loadingStatus: false,
    }
  },  
  methods: {
    search() {
      this.loadingStatus = true;
      let modUrl = new Url(this.query);
      if (modUrl.host != 'modrinth.com') {
        this.queryModrinthAPI(this.query);
      } else {
        this.queryModrinthAPI(modUrl.pathname.split('/')[2]);
      }
      this.query = '';
    },
    importModList() {
      this.loadingStatus = true;
      for (let mod of this.importQuery.split(',')) {
        this.queryModrinthAPI(mod);
      }
    },
    async queryModrinthAPI(modName) {
      let response;
      try {
        response = await axios.get(`https://api.modrinth.com/api/v1/mod/${modName}`)
      } catch(e) {
        this.$notify({
          title: 'Failure',
          message: `The mod ${this.query} is not found.`,
          type: 'error',
          duration: 2500,
          customClass: 'errorNotification'
        })
        this.loadingStatus = false;
      }

      let queryData = await response.data;
      if (!this.exportData.includes(queryData.id)) {
        const dateOptions = { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' }

        let newMod = {
          id: queryData.id,
          name: queryData.title,
          desc: queryData.description,
          followers: queryData.followers,
          downloads: queryData.downloads,
          icon_url: queryData.icon_url,
          published: new Date(queryData.published).toLocaleDateString(undefined, dateOptions),
          updated: new Date(queryData.updated).toLocaleDateString(undefined, dateOptions),
          client: this.formatTerm(queryData.client_side),
          server: this.formatTerm(queryData.server_side),
          categories: this.formatTerm(queryData.categories),
          license: queryData.license.id,
        } 

        if (queryData.versions) {
          newMod = {
            ...newMod,
            ...await this.queryModrinthVersionAPI([queryData.title, queryData.versions.slice(-1)[0]])
          }
        }

        console.log(newMod);
        this.exportData.push(queryData.id);
        this.displayedData.push(newMod);
        this.loadingStatus = false;
      } else {
          this.$notify({
            title: 'Failure',
            message:  `The mod ${queryData.title} has already been added to the list.`,
            type: 'error',
            duration: 2500,
            customClass: 'errorNotification'
          });
          this.loadingStatus = false;
        }
      }, async queryModrinthVersionAPI(versionRequest) {
        let response;
        try {
          response = await axios.get(`https://api.modrinth.com/api/v1/version/${versionRequest[1]}`)
          let queryData = await response.data;

          return ({  
            game_versions: queryData.game_versions.join(', '),
            version_type: this.formatTerm(queryData.version_type),
            loader: this.formatTerm(queryData.loaders),
            filename: queryData.files[0].filename,
            fileUrl: queryData.files[0].url,
          });
        } catch(e) {
          this.$notify({
            title: 'Failure',
            message: `Error getting versions for ${versionRequest[0]}`,
            type: 'error',
            duration: 2500,
            customClass: 'errorNotification'
          })
          this.loadingStatus = false;
        }
      },
      handleDelete(index) {
        this.exportData.forEach((item) => {
          if (item == this.displayedData[index].id) {
            this.exportData.splice(index, 1)
          }
        });
        this.displayedData.splice(index, 1);
      }, formatTerm(word) {
        if (Array.isArray(word)) {
          word.forEach((item, index) => {
            word[index] = item[0].toUpperCase() + item.slice(1);
          })

          return word.join(', ');
        } else {
          return (word[0].toUpperCase() + word.slice(1)).replace('_',' ');
        }
      }, handleSelectChange(value) {
          let checkedCount = value.length;
          this.checkAll = checkedCount === this.columnOptions.length;
          this.isIndeterminate = checkedCount > 0 && checkedCount < this.columnOptions.length;
      }, handleSelectAllChange(value) {
          this.columns = value ? defaultOptions : [];
          this.isIndeterminate = false;
      }
  }
}

</script>

<style>
body, html {
  background-color: white;
}

.el-table-column {
 color: red;
}

.el-button {
  background: #fca311;
  color: #F7FAFF;
  border-color:white;
}

.el-button:hover {
  background: #E5E5E5;
  color: #050C1C;
}

.flexContainer {
	display: flex;
	width: 50%;
	margin: auto;
}

.errorNotification {
	font-family: sans-serif;
}

.submitBtn { 
  width: 10rem;
  margin-left: 2rem;
  background-color: '#fca311';
}

.el-checkbox-group {
  color: white;
}

.el-checkbox-group.__inner .checked {
  color: white;
}

.el-checkbox.__inner .checked {
  color: white;
}

.el-checkbox.fill {
  color: white;
}
</style>
