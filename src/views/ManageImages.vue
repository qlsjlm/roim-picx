<template>
	<div class="mx-auto max-w-6xl my-4 px-4 relative">
		<loading-overlay :loading="loading" />

		<div class="flex items-center justify-between mb-4">
			<div>
				<div class="text-gray-800 text-lg">管理图片</div>
				<div class="text-sm text-gray-500">
					已上传 {{ uploadedImages.length }} 张图片，共 {{ formatBytes(imagesTotalSize) }}
				</div>
			</div>
      <div class="flex items-center justify-start">
        <font-awesome-icon :icon="faFolderPlus" class="text-xl cursor-pointer text-3xl text-amber-300 mr-2" @click="addFolder" />
        <font-awesome-icon
            :icon="faRedoAlt"
            class="text-xl cursor-pointer text-indigo-400"
            @click="listImages"
        />
      </div>
		</div>

    <div class="my-2 flex items-center justify-start flex-wrap">
      <div v-for="it in prefixes" class="px-4 py-2 items-center flex rounded-lg bg-white shadow-md cursor-pointer mx-1" @click="changeFolder(it)">
        <font-awesome-icon :icon="faFolder" class="text-3xl text-amber-500" />
        <span v-if="it !== '/'" class="pl-2 text-gray-600"> {{ it.replace("/", "") }}</span>
        <span v-else class="pl-2 text-gray-600"> {{ it }}</span>
      </div>
    </div>

		<div class="grid gap-2 lg:gap-4 lg:grid-cols-4 grid-cols-2">
			<transition-group name="el-fade-in-linear">
				<div
					class="col-span-1 md:col-span-1"
					v-for="item in uploadedImages"
					:key="item.url"
				>
					<image-box
						:src="item.url"
						:name="item.key"
            :size="item.size"
						@delete="deleteImage(item.key)"
						mode="uploaded"
					/>
					
          <!-- 复制链接按钮，悬停时显示多个链接格式 -->
          <div class="mt-2 relative">
            <button
              @mouseenter="showFormats = item.key"
              @mouseleave="showFormats = null"
              class="text-sm text-blue-500 hover:underline"
            >
              链接
            </button>

            <!-- 鼠标悬停时显示不同格式的链接 -->
            <div v-if="showFormats === item.key" class="absolute bg-white border p-2 shadow-md mt-2 w-48">
              <p><strong>Markdown 格式：</strong></p>
              <div class="cursor-pointer" @click="copyLink(`![${item.name}](${item.url})`)">
                ![{{ item.name }}]({{ item.url }})
              </div>
              <p><strong>HTML 格式：</strong></p>
              <div class="cursor-pointer" @click="copyLink(`<img src='${item.url}' alt='${item.name}' />`)">
                <img :src="item.url" :alt="item.name" />
              </div>
              <p><strong>图片 URL：</strong></p>
              <div class="cursor-pointer" @click="copyLink(item.url)">
                {{ item.url }}
              </div>
            </div>
          </div>
				</div>
			</transition-group>
		</div>
	</div>
</template>

<script setup lang="ts">
import { requestListImages, requestDeleteImage, createFolder } from '../utils/request'
import LoadingOverlay from '../components/LoadingOverlay.vue'
import formatBytes from '../utils/format-bytes'
import { computed, onMounted, ref } from 'vue'
import type { ImgItem, ImgReq, Folder } from '../utils/types'
import ImageBox from '../components/ImageBox.vue'
import { ElMessageBox, ElMessage } from 'element-plus'
import { faRedoAlt, faFolder, faFolderPlus } from '@fortawesome/free-solid-svg-icons'
import {FontAwesomeIcon} from "@fortawesome/vue-fontawesome";

const loading = ref(false)
const delimiter = ref('/')
const uploadedImages = ref<ImgItem[]>([])
const prefixes = ref<String[]>([])
const imagesTotalSize = computed(() =>
    uploadedImages.value.reduce((total, item) => total + item.size, 0)
)

const showFormats = ref<string | null>(null)  // 用来显示链接格式的控制

const changeFolder = (path : string) => {
  console.log(path)
  delimiter.value = path
  listImages()
}

const addFolder = () => {
  ElMessageBox.prompt('请输入目录名称，仅支持英文名称', '新增目录', {
    confirmButtonText: '创建',
    cancelButtonText: '取消',
    inputPattern: /^[A-Za-z_]+$/,
    inputErrorMessage: '无效的目录名称',
  }).then(({ value }) => {
    loading.value = true
    createFolder(<Folder> {
      name: value
    }).then((res) => {
      console.log(res)
      ElMessage.success('文件见创建成功')
      listImages()
    }).catch(() => {
      ElMessage.error('文件见创建失败')
    }).finally(() => {
      loading.value = false
    })
  }).catch(() => {})
}

const listImages = () => {
	loading.value = true
	requestListImages(<ImgReq> {
    limit: 100,
    delimiter: delimiter.value
  }).then((data) => {
    uploadedImages.value = data.list
    if (data.prefixes && data.prefixes.length) {
      prefixes.value = data.prefixes
      if (delimiter.value !== '/') {
        prefixes.value = ['/', ...data.prefixes]
      }
    } else {
      prefixes.value = ['/']
    }
  }).catch(() => {})
		.finally(() => {
			loading.value = false
		})
}

onMounted(() => {
	listImages()
})

const deleteImage = (src: string) => {
	requestDeleteImage({
    keys: src
  }).then((res) => {
		uploadedImages.value = uploadedImages.value.filter((item) => item.key !== res)
	})
}

const copyLink = (link: string) => {
  navigator.clipboard.writeText(link).then(() => {
    ElMessage.success('复制成功！')
  }).catch(() => {
    ElMessage.error('复制失败！')
  })
}
</script>

<style scoped>
/* 可以根据需要添加一些样式 */
</style>
