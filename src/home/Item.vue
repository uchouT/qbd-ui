<template>
    <el-card shadow="hover">
        <template #header>
            <div class="task-name">{{ props.taskItem.name }}</div>
        </template>
        <el-row class="task-status" justify="space-between">
            <el-col :span="20"
                style="display: flex; flex-direction: column; gap: 10px; justify-content: space-between;">
                <div class="task-progress">
                    分片: {{ props.taskItem.state.current_part_num + 1 }}/{{ props.taskItem.total_part_num }}
                </div>
                <el-space class="task-status-tag">
                    <el-tag :type="getStatusType(props.taskItem.state.status)" size="small" style="text-align: right;">
                        {{ getStatusText(props.taskItem.state.status) }}
                    </el-tag>
                    <el-tag v-if="props.taskItem.state.is_seeding" type="primary" size="small">
                        做种中
                    </el-tag>
                </el-space>
                <div><el-progress :text-inside="true" :show-text="false"
                        :status="getProcessStatus(props.taskItem.state.status)"
                        :percentage="(props.taskItem.state.progress * 100).toFixed(2)">
                    </el-progress></div>

            </el-col>

            <el-col :span="2" style="display: flex; flex-direction: column; gap: 10px; justify-content: space-between;">
                <el-button plain size="small" v-if="props.taskItem.state.status === 'Paused'"
                    @click="startTask(props.taskItem.hash)" type="success" :loading="actionLoading" icon="VideoPlay">
                </el-button>
                <el-button plain size="small" v-if="props.taskItem.state.status === 'Downloading'"
                    @click="stopTask(props.taskItem.hash)" type="warning" :loading="actionLoading" icon="VideoPause">
                </el-button>
                <el-button plain size="small" @click="deleteTask(props.taskItem.hash)" type="danger"
                    :loading="actionLoading" icon="Delete">
                </el-button>
                <el-button plain size="small" @click="showDetail = !showDetail" icon="InfoFilled"
                    type="info"></el-button>
            </el-col>
        </el-row>



        <el-dialog v-model="showDetail">
            <div class="task-details" style="font-size: 14px;">
                <el-row>
                    <span class="label">种子名称:</span>
                    <span> {{ props.taskItem.name }}</span>
                </el-row>
                <el-row>
                    <span class="label">保存路径:</span>
                    <span>{{ props.taskItem.save_path }}</span>
                </el-row>
                <el-row>
                    <span class="label">上传路径:</span>
                    <span>{{ props.taskItem.upload_path }}</span>
                </el-row>
                <el-row>
                    <span class="label">上传方式:</span>
                    <span>{{ props.taskItem.uploader.type }}</span>
                </el-row>
                <el-row>
                    <span class="label">最大容量:</span>
                    <span>{{ readableSize() }}</span>
                </el-row>
            </div>
            <div class="error-info" v-if="props.taskItem.state.status === 'Error'">
                <hr />
                <el-row>
                    <span class="label">发生时间</span>
                    <span>{{ props.taskItem.error_info.timestamp }}</span>
                </el-row>
                <el-row>
                    <span class="label">错误类型:</span>
                    <el-tag type="danger">{{ props.taskItem.error_info.kind }}</el-tag>
                </el-row>
                <p>{{ errorTipsMap[props.taskItem.error_info.kind] }}</p>
                <p>更多细节请查看日志。</p>
                <el-space class="error-manipulation">
                    <el-button icon="VideoPlay" plain type="warning" @click="resumeTask(props.taskItem.hash)"
                        :loading="actionLoading">尝试恢复</el-button>
                    <el-button icon="VideoPlay" plain type="danger" :loading="actionLoading"
                        @click="skipTask(props.taskItem.hash)"
                        v-if="errorAllowForceSkipMap[props.taskItem.error_info.kind]">强制继续</el-button>
                </el-space>
            </div>
        </el-dialog>
    </el-card>
</template>

<script setup>
import { ref } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import api from '../api'

const actionLoading = ref(false)
const props = defineProps(['taskItem'])
const emit = defineEmits(['refresh'])
const showDetail = ref(false)
// 状态映射
const statusTextMap = {
    'OnTask': '上传中',
    'Finished': '上传完成',
    'Downloaded': '下载完成',
    'Downloading': '下载中',
    'Done': '全部完成',
    'Error': '错误',
    'Paused': '已暂停'
}
const errorTipsMap = {
    'TorrentNotFound': "qBittorrent 种子任务缺失。「尝试恢复」或「强制继续」将从缓存中重新添加任务。",
    'RuntimeUpload': "上传过程中遇到问题。「尝试恢复」会重新提交一个上传任务。你也可以手动完成上传后「强制继续」， 这会直接开启下一个分片任务。",
    'LaunchUpload': "提交上传任务失败，请检查上传器的配置。「尝试恢复」会尝试重新提交上传任务",
    'Download': "qBittorrent 下载过程中发生错误。请手动确认 qBittorrent 状态后「尝试恢复」",
    'AddNextPart': "添加下一个分片失败，可能是 torrent 缓存丢失。"
}

const errorAllowForceSkipMap = {
    'TorrentNotFound': true,
    'RuntimeUpload': true,
}
const readableSize = () => {
    let size = props.taskItem.max_size / 1024 / 1024; // turn to MB
    if (size < 1024) {
        return `${size} MB`;
    }
    return `${size / 1024} GB`;
}
const statusTypeMap = {
    'OnTask': 'warning',
    'Finished': 'success',
    'Downloaded': 'success',
    'Downloading': 'primary',
    'Done': 'success',
    'Error': 'danger',
    'Paused': 'info'
}

const processStatusMap = {
    'OnTask': 'warning',
    'Done': 'success',
    'Error': 'exception',
    'Paused': 'warning',
    'Downloaded': 'success'
}

const getProcessStatus = (status) => {
    return processStatusMap[status];
}
// 方法
const getStatusText = (status) => {
    return statusTextMap[status] || status
}

const getStatusType = (status) => {
    return statusTypeMap[status] || 'info'
}


// 开始任务
const startTask = async (hash) => {
    await task_(hash, 'start')
}

// 停止任务
const stopTask = async (hash) => {
    await task_(hash, 'stop')
}

const resumeTask = async (hash) => {
    await startTask(hash);
    showDetail.value = false;
}

const skipTask = async (hash) => {
    await task_(hash, 'start', true)
    showDetail.value = false;
}

const task_ = async (hash, type, skip = false) => {
    actionLoading.value = true
    await api.put(`/api/task?type=${type}&hash=${hash}&skip=${skip}`)
        .then(async () => {
            emit('refresh')
            ElMessage.success(`task ${type} success`)
        })
        .finally(() => {
            actionLoading.value = false
        })
}

// 删除任务
const deleteTask = async (hash) => {

    await ElMessageBox.confirm(
        '确定要删除这个任务吗？此操作不可恢复。',
        '确认删除',
        {
            confirmButtonText: '确定',
            cancelButtonText: '取消',
            type: 'warning',
        }
    )

    actionLoading.value = true
    await api.del(`/api/task?hash=${hash}`)
        .then(async () => {
            emit('refresh')
            ElMessage.success('Task delete success')
        })
        .finally(() => {
            actionLoading.value = false
        })

}
</script>

<style scoped>
.el-button {
    margin: 0;
}

.task-name {
    margin: 0 0 8px 0;
    font-size: 16px;
    font-weight: 600;
    color: var(--el-text-color-primary);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.task-progress {
    font-size: 12px;
    color: var(--el-text-color-secondary);
}

.label {
    min-width: 80px;
    color: var(--el-text-color-secondary);
    font-weight: 500;
}
</style>