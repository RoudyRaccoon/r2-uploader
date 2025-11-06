<template>
  <div class="file-browser">
    <!-- Toolbar: Breadcrumb + View Toggle + Upload -->
    <el-row align="middle" class="toolbar" :gutter="12" style="margin-bottom:12px">
      <el-col :span="18">
        <el-breadcrumb separator="/" class="breadcrumb">
          <el-breadcrumb-item v-for="(crumb, idx) in breadcrumbs" :key="idx">
            <a href="#" @click.prevent="goToBreadcrumb(idx)">{{ crumb || '/' }}</a>
          </el-breadcrumb-item>
        </el-breadcrumb>
      </el-col>

      <el-col :span="6" style="text-align:right">
        <el-button-group>
          <el-tooltip content="Details view" placement="bottom">
            <el-button :type="viewMode === 'details' ? 'primary' : 'default'" @click="viewMode = 'details'">
              <i class="el-icon-s-data"></i>
            </el-button>
          </el-tooltip>

          <el-tooltip content="Icons view" placement="bottom">
            <el-button :type="viewMode === 'icons' ? 'primary' : 'default'" @click="viewMode = 'icons'">
              <i class="el-icon-picture"></i>
            </el-button>
          </el-tooltip>

          <el-upload
            class="upload-btn"
            :http-request="customUpload"
            :multiple="true"
            :show-file-list="false"
            drag
            :data="uploadExtraData"
            :before-upload="onBeforeUpload"
          >
            <el-button size="mini">Upload</el-button>
          </el-upload>
        </el-button-group>
      </el-col>
    </el-row>

    <!-- Quick path + refresh -->
    <el-row style="margin-bottom:8px" align="middle">
      <el-col :span="20">
        <el-input v-model="currentPath" size="small" @keyup.enter.native="listCurrentPath" clearable>
          <template #append>
            <el-button icon="el-icon-refresh" @click="listCurrentPath" size="small"></el-button>
          </template>
        </el-input>
      </el-col>
      <el-col :span="4" style="text-align:right">
        <small>{{ items.length }} items</small>
      </el-col>
    </el-row>

    <!-- Views -->
    <div v-if="viewMode === 'details'">
      <el-table
        :data="sortedItems"
        style="width: 100%"
        @row-dblclick="onRowDblClick"
        :default-sort = "{prop: 'is_dir', order: 'descending'}"
      >
        <el-table-column prop="name" label="Name" min-width="240">
          <template #default="{row}">
            <div class="name-cell">
              <i :class="iconClass(row)"></i>
              <img v-if="showThumbnail(row)" :src="thumbnailUrl(row)" class="thumb small"/>
              <span class="name-text" @dblclick.stop="onRowDblClick(row)">{{ displayName(row) }}</span>
            </div>
          </template>
        </el-table-column>

        <el-table-column prop="size" label="Size" width="120">
          <template #default="{row}">{{ row.is_dir ? '-' : humanSize(row.size) }}</template>
        </el-table-column>

        <el-table-column prop="last_modified" label="Modified" width="180">
          <template #default="{row}">{{ formatDate(row.last_modified) }}</template>
        </el-table-column>

        <el-table-column label="Actions" width="120">
          <template #default="{row}">
            <el-button size="mini" type="text" @click.stop="download(row)">Open</el-button>
            <el-dropdown trigger="click" @command="onCommand(row, $event)">
              <el-button size="mini">•••</el-button>
              <el-dropdown-menu slot="dropdown">
                <el-dropdown-item command="download">Download</el-dropdown-item>
                <el-dropdown-item command="delete">Delete</el-dropdown-item>
                <el-dropdown-item command="copy">Copy URL</el-dropdown-item>
              </el-dropdown-menu>
            </el-dropdown>
          </template>
        </el-table-column>
      </el-table>
    </div>

    <div v-else class="icons-view">
      <el-row :gutter="16">
        <el-col :span="24">
          <div class="icon-grid">
            <div v-for="item in sortedItems" :key="item.key" class="icon-tile" @dblclick="onRowDblClick(item)">
              <el-card :body-style="{ padding: '8px' }" class="tile-card">
                <div class="tile-content">
                  <div class="tile-icon">
                    <img v-if="showThumbnail(item)" :src="thumbnailUrl(item)" class="thumb big"/>
                    <i v-else :class="iconClass(item) + ' big-icon'"></i>
                  </div>
                  <div class="tile-name" :title="item.name">{{ displayName(item) }}</div>
                  <div class="tile-meta">{{ item.is_dir ? 'Folder' : humanSize(item.size) }}</div>
                </div>
              </el-card>
            </div>
          </div>
        </el-col>
      </el-row>
    </div>

  </div>
</template>

<script>
import { ref, reactive, computed, onMounted } from 'vue';
import { ElMessage, ElMessageBox } from 'element-plus';

export default {
  name: 'FileBrowserDualView',
  props: {
    // root API endpoints used by r2-uploader default backend. You can override if your backend differs.
    listEndpoint: { type: String, default: '/api/list' },
    downloadEndpoint: { type: String, default: '/api/download' },
    uploadEndpoint: { type: String, default: '/api/upload' },
    thumbnailEndpoint: { type: String, default: '/api/thumbnail' },
    initialPath: { type: String, default: '' },
    apiKeyHeader: { type: String, default: 'x-api-key' },
    apiKeyValue: { type: String, default: '' }
  },
  setup(props, { emit }) {
    const currentPath = ref(props.initialPath || '');
    const items = ref([]); // array of { key, name, is_dir, size, last_modified, url }
    const viewMode = ref('details');

    const breadcrumbs = computed(() => {
      if (!currentPath.value) return [''];
      const parts = currentPath.value.split('/').filter(Boolean);
      const crumbs = [''];
      let p = '';
      parts.forEach(part => {
        p = p + '/' + part;
        crumbs.push(p);
      });
      return crumbs;
    });

    const sortedItems = computed(() => {
      const copy = items.value.slice();
      copy.sort((a, b) => {
        if (a.is_dir && !b.is_dir) return -1;
        if (!a.is_dir && b.is_dir) return 1;
        // folders/files both; sort alphabetically
        return a.name.localeCompare(b.name, undefined, { sensitivity: 'base' });
      });
      return copy;
    });

    function authHeaders() {
      const headers = new Headers();
      if (props.apiKeyValue) headers.set(props.apiKeyHeader, props.apiKeyValue);
      return headers;
    }

    async function listCurrentPath() {
      try {
        const url = new URL(props.listEndpoint, location.origin);
        url.searchParams.set('prefix', currentPath.value || '');
        const res = await fetch(url.toString(), { headers: authHeaders() });
        if (!res.ok) throw new Error(await res.text());
        const data = await res.json();
        // Expect data.items = [{ key, name, is_dir, size, last_modified, url? }]
        items.value = (data.items || []).map(it => ({ ...it }));
      } catch (err) {
        ElMessage.error('Failed to list files: ' + err.message);
        console.error(err);
      }
    }

    function goToBreadcrumb(idx) {
      if (idx === 0) {
        currentPath.value = '';
      } else {
        const parts = currentPath.value.split('/').filter(Boolean);
        currentPath.value = parts.slice(0, idx).join('/');
      }
      listCurrentPath();
    }

    function displayName(item) {
      if (item.is_dir) return item.name.replace(/\/$/, '');
      // show only basename
      return item.name.split('/').pop();
    }

    function humanSize(size) {
      if (size == null) return '';
      const thresh = 1024;
      if (Math.abs(size) < thresh) return size + ' B';
      const units = ['KB','MB','GB','TB'];
      let u = -1;
      do {
        size /= thresh;
        ++u;
      } while(Math.abs(size) >= thresh && u < units.length - 1);
      return size.toFixed(1) + ' ' + units[u];
    }

    function formatDate(iso) {
      if (!iso) return '';
      const d = new Date(iso);
      return d.toLocaleString();
    }

    function iconClass(item) {
      if (item.is_dir) return 'el-icon-folder';
      // basic extension detection
      const ext = (item.name.split('.').pop() || '').toLowerCase();
      if (['png','jpg','jpeg','webp','gif','bmp'].includes(ext)) return 'el-icon-picture';
      if (['mp4','mkv','webm','mov','avi'].includes(ext)) return 'el-icon-video-camera';
      if (['mp3','wav','ogg'].includes(ext)) return 'el-icon-music';
      if (['zip','tar','gz','rar'].includes(ext)) return 'el-icon-files';
      if (['pdf'].includes(ext)) return 'el-icon-document';
      return 'el-icon-document';
    }

    function showThumbnail(item) {
      if (item.is_dir) return false;
      const ext = (item.name.split('.').pop() || '').toLowerCase();
      return ['png','jpg','jpeg','webp','gif','bmp'].includes(ext);
    }

    function thumbnailUrl(item) {
      if (item.url) return item.url; // if backend provided signed URL
      // try thumbnail endpoint
      const url = new URL(props.thumbnailEndpoint, location.origin);
      url.searchParams.set('key', item.key || item.name);
      return url.toString();
    }

    async function onRowDblClick(row) {
      if (!row) return;
      if (row.is_dir) {
        // navigate into folder
        currentPath.value = (currentPath.value ? currentPath.value + '/' : '') + row.name.replace(/\/$/, '');
        await listCurrentPath();
      } else {
        // open / download
        download(row);
      }
    }

    async function download(row) {
      // if backend provides direct URL, open it. Otherwise call download endpoint.
      try {
        if (row.url) {
          window.open(row.url, '_blank');
          return;
        }
        const url = new URL(props.downloadEndpoint, location.origin);
        url.searchParams.set('key', row.key || row.name);
        const res = await fetch(url.toString(), { headers: authHeaders() });
        if (!res.ok) throw new Error(await res.text());
        const blob = await res.blob();
        const a = document.createElement('a');
        a.href = URL.createObjectURL(blob);
        a.download = displayName(row);
        document.body.appendChild(a);
        a.click();
        a.remove();
      } catch (err) {
        ElMessage.error('Download failed: ' + err.message);
      }
    }

    async function onCommand(row, cmd) {
      if (cmd === 'download') return download(row);
      if (cmd === 'delete') {
        try {
          await ElMessageBox.confirm(`Delete ${row.name}?`, 'Confirm', { type: 'warning' });
          // call delete endpoint
          const url = new URL(props.listEndpoint, location.origin);
          url.searchParams.set('key', row.key || row.name);
          const res = await fetch(url.toString(), { method: 'DELETE', headers: authHeaders() });
          if (!res.ok) throw new Error(await res.text());
          ElMessage.success('Deleted');
          listCurrentPath();
        } catch (err) {
          if (err !== 'cancel') ElMessage.error('Delete failed: ' + err.message);
        }
      }
      if (cmd === 'copy') {
        const url = row.url || (location.origin + props.downloadEndpoint + '?key=' + encodeURIComponent(row.key || row.name));
        await navigator.clipboard.writeText(url);
        ElMessage.success('Copied URL');
      }
    }

    // Custom upload hook used by el-upload
    async function customUpload({ file, onSuccess, onError, onProgress }) {
      try {
        const form = new FormData();
        form.append('file', file);
        form.append('path', currentPath.value || '');
        const xhr = new XMLHttpRequest();
        xhr.open('POST', props.uploadEndpoint, true);
        // headers
        if (props.apiKeyValue) xhr.setRequestHeader(props.apiKeyHeader, props.apiKeyValue);
        xhr.upload.onprogress = (e) => {
          if (e.lengthComputable && onProgress) onProgress({ percent: (e.loaded / e.total) * 100 });
        };
        xhr.onload = () => {
          if (xhr.status >= 200 && xhr.status < 300) {
            try { onSuccess(JSON.parse(xhr.responseText)); } catch(e) { onSuccess(xhr.responseText); }
            listCurrentPath();
          } else {
            onError(new Error('Upload failed: ' + xhr.statusText));
          }
        };
        xhr.onerror = () => onError(new Error('Network error'));
        xhr.send(form);
      } catch (err) {
        onError(err);
      }
    }

    function uploadExtraData() {
      return { path: currentPath.value || '' };
    }

    function onBeforeUpload(file) {
      // you can add client-side checks here
      return true;
    }

    onMounted(() => {
      listCurrentPath();
    });

    return {
      currentPath,
      items,
      viewMode,
      breadcrumbs,
      sortedItems,
      displayName,
      humanSize,
      formatDate,
      iconClass,
      thumbnailUrl,
      showThumbnail,
      listCurrentPath,
      goToBreadcrumb,
      onRowDblClick,
      download,
      onCommand,
      customUpload,
      uploadExtraData,
      onBeforeUpload
    };
  }
};
</script>

<style scoped>
.file-browser { padding: 8px; }
.toolbar { margin-bottom: 12px; }
.name-cell { display:flex; align-items:center; gap:8px }
.name-cell .thumb.small { width:28px; height:28px; object-fit:cover; border-radius:4px }
.thumb.big { width:64px; height:64px; object-fit:cover; border-radius:6px }
.tile-card { width:120px; text-align:center }
.icon-grid { display:flex; flex-wrap:wrap; gap:12px }
.icon-tile { width:120px }
.tile-content { display:flex; flex-direction:column; align-items:center; gap:6px }
.big-icon { font-size:40px; }
.tile-name { white-space:nowrap; overflow:hidden; text-overflow:ellipsis; max-width:110px }
.tile-meta { font-size:11px; color:var(--el-text-color-secondary) }
</style>
