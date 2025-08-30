<template>
    <TorrentContent ref="torrentContent" :hash="taskData.torrent_res.hash"
        v-model:selected_file_index="taskData.selected_file_index"></TorrentContent>
    <el-form ref="formRef" :model="taskData" :rules="rules" label-position="right" label-width="140px">
        <el-form-item label="种子名称">
            <el-text>{{ taskData.torrent_res.torrent_name }}</el-text>
        </el-form-item>
        <el-form-item label="上传路径">
            <el-input v-model="taskData.upload_path" :placeholder="upload_path" />
        </el-form-item>
        <el-form-item label="文件大小限制" prop="max_size">
            <el-input-number v-model="taskData.max_size" :precision=0 :min="1" :max="999">
                <template #suffix>GB</template>
            </el-input-number>
        </el-form-item>
        <span>
            <el-form-item label="自定义内容">
                <el-space size="large">
                    <el-switch v-model="taskData.custom_content"></el-switch>
                    <el-button icon="Files" :disabled="!taskData.custom_content"
                        @click="torrentContent?.show()"></el-button>
                </el-space>
            </el-form-item>
        </span>
        <el-form-item label="上传工具" prop="upload_type">
            <el-radio-group v-model="taskData.upload_type.type">
                <el-radio value="Rclone">Rclone</el-radio>
                <!-- <el-radio value="alist">Alist</el-radio> -->
            </el-radio-group>
        </el-form-item>
        <el-form-item label="做种时间限制">
            <el-input-number v-model="taskData.seeding_time_limit" :placeholder="-2">
                <template #suffix>分钟</template>
            </el-input-number>
        </el-form-item>
        <el-form-item label="分享率限制">
            <el-input-number v-model="taskData.ratio_limit" :precision="2" :placeholder="-2" :step="0.1" />
        </el-form-item>
    </el-form>
    <div class="warning-box flex-center">
        <p><strong>重要提醒：</strong>务必设置做种限制，否则分片任务将永远不会结束;<br>
            <small>-2 表示使用 qBittorrent 的全局设置</small>
        </p>
    </div>
    <div style="height: 20px;"></div>
    <div class="form-actions" style="text-align: right;">
        <el-button @click="emit('cancel')">取消</el-button>
        <el-button type="primary" :loading="loading" @click="handleSubmit">
            {{ loading ? '处理中...' : '完成' }}
        </el-button>
    </div>
</template>

<script setup>
import { computed, ref } from 'vue';
import { ElMessage } from 'element-plus';
import CONFIG from '../config';
import api from '../api';
import TorrentContent from './TorrentContent.vue';
const upload_path = computed(() => {
    return CONFIG.value.default_upload_path || '请输入上传路径';
});
const torrentContent = ref();
const taskData = defineModel('taskData')
const emit = defineEmits(['ok', 'cancel'])
const loading = ref(false)
const formRef = ref(null)
const rules = ref({
    upload_type: [{ required: true, message: '请选择上传工具', trigger: 'change' }],
    max_size: [{ required: true, message: '请输入文件大小限制', trigger: 'blur' }]
})

const checkUploader = async () => {
    let test_req = {
        test_type: taskData.value.upload_type.type,
        host: '',
        username: '',
        password: '',
    }
    if (test_req.test_type === 'Rclone') {
        test_req.host = CONFIG.value.rclone_host || '';
        test_req.username = CONFIG.value.rclone_username || '';
        test_req.password = CONFIG.value.rclone_password || '';
    }
    return await api.post('api/test', test_req)
}
const handleSubmit = async () => {
    if (!formRef.value) return
    loading.value = true
    try {
        await formRef.value.validate()
    } catch (error) {
        ElMessage.warning('请填写完整的信息')
        return
    } finally {
        loading.value = false
    }
    try {
        await checkUploader()
    } catch (error) {
        ElMessage.error('上传工具测试失败，请检查配置')
        return
    } finally {
        loading.value = false
    }
    emit('ok', () => {
        loading.value = false
    })
}

</script>

<style scoped>
.warning-box {
    background: rgba(255, 193, 7, 0.1);
    border: 1px solid rgba(255, 193, 7, 0.3);
    line-height: normal;
    border-radius: 6px;
    font-size: 16px;
}
</style>