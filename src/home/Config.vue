<template>
    <el-dialog v-model="dialogVisible" title="设置" :width="dialogWidth" class="config-dialog">
        <div v-loading="loading">
            <el-tabs v-model="activeName" type="card" class="demo-tabs">
                <el-tab-pane label="Basic" name="first">
                    <el-scrollbar>
                        <el-form label-position="right" label-width="140px">
                            <el-form-item label="QB Host">
                                <el-input v-model="config.qb_host" placeholder="http://localhost:8080" />
                            </el-form-item>
                            <el-form-item label="QB 用户名">
                                <el-input v-model="config.qb_username" placeholder="admin" />
                            </el-form-item>
                            <el-form-item label="QB 密码">
                                <el-input v-model="config.qb_password" type="password" placeholder="password" />
                            </el-form-item>
                            <el-form-item>
                                <el-button @click="test('qb')" bg text :loading="qbTestLoading" icon="Odometer">测试
                                </el-button>
                            </el-form-item>
                            <el-form-item label="only inner IP">
                                <el-switch v-model="config.is_only_inner_ip" />
                            </el-form-item>
                            <el-form-item label="allow multi-login">
                                <el-switch v-model="config.multi_login" />
                            </el-form-item>
                            <el-form-item label="Username">
                                <el-input v-model="config.account.username" placeholder="admin" />
                            </el-form-item>
                            <el-form-item label="Password">
                                <el-input v-model="config.account.password" type="password" placeholder="password" />
                            </el-form-item>
                            <el-form-item label="默认分享率">
                                <el-input-number v-model="config.default_ratio_limit" placeholder="-2" />
                            </el-form-item>
                            <el-form-item label="默认做种时间">
                                <el-input-number v-model="config.default_seeding_time_limit" placeholder="-2">
                                    <template #suffix>分钟</template>
                                </el-input-number>
                            </el-form-item>
                        </el-form>
                    </el-scrollbar>
                </el-tab-pane>
                <el-tab-pane label="Uploader" name="second">
                    <el-scrollbar>

                        <el-form label-position="right" label-width="140px">
                            <el-form-item label="Rclone Host">
                                <el-input v-model="config.rclone_host" placeholder="http://localhost:5572" />
                            </el-form-item>
                            <el-form-item label="Rclone Username">
                                <el-input v-model="config.rclone_username" placeholder="admin" />
                            </el-form-item>
                            <el-form-item label="Rclone Password">
                                <el-input v-model="config.rclone_password" type="password" placeholder="secret" />
                            </el-form-item>
                            <el-form-item>
                                <el-button @click="test('Rclone')" bg text :loading="RcloneTestLoading"
                                    icon="Odometer">测试
                                </el-button>
                            </el-form-item>
                            <el-form-item label="默认下载路径">
                                <el-input v-model="config.default_save_path" placeholder="/downloads" />
                            </el-form-item>
                            <el-form-item label="默认上传路径">
                                <el-input v-model="config.default_upload_path" placeholder="/uploads" />
                            </el-form-item>
                        </el-form>
                    </el-scrollbar>
                </el-tab-pane>
                <el-tab-pane label="About" name="third" class="flex-center-column">
                    <h2 style="color: var(--el-color-primary)">v{{ useLocalStorage('version', '0.0.0') }}</h2>
                    <img src="../../public/icon.png" alt="icon" style="width: 80px;" />
                    <div class="description" style="max-width:  60%;">
                        <h3>qb-downloader</h3>
                    </div>
                    <el-space>
                        <a href="https://t.me/uchouT" target="_blank" style="color: inherit">
                            <i class="fa-brands fa-telegram fa-xl"></i></a>
                        <a href="https://github.com/uchouT/qb-downloader" style="color: inherit;" target="_blank">
                            <i class="fa-brands fa-github fa-xl"></i></a>
                    </el-space>
                </el-tab-pane>
            </el-tabs>
        </div>
        <template #footer>
            <div class="dialog-footer">
                <el-button @click="dialogVisible = false">取消</el-button>
                <el-button type="primary" :loading="loading" @click="saveConfig">保存配置</el-button>
            </div>
        </template>
    </el-dialog>
</template>

<script setup>
import { ref, computed } from 'vue';
import { useWindowSize, useLocalStorage } from '@vueuse/core'
import api from '../api';

import { ElMessage } from 'element-plus';
import CONFIG from '../config';

const config = ref();
const { width } = useWindowSize()

// 响应式对话框宽度
const dialogWidth = computed(() => {
    if (width.value <= 768) {
        return '95%'
    } else if (width.value <= 1024) {
        return '70%'
    } else {
        return '50%'
    }
})

const saveConfig = () => {
    loading.value = true
    Object.assign(CONFIG.value, config.value)

    api.post('api/config', config.value)
        .then(res => {
            ElMessage.success(res.message)
            dialogVisible.value = false
        })
        .finally(() => {
            config.value = null
            loading.value = false
        })
}
const qbTestLoading = ref(false)
const RcloneTestLoading = ref(false)
const dialogVisible = ref(false)
const activeName = ref('first')


const loading = ref(false)

const show = () => {
    dialogVisible.value = true;
    loading.value = true;
    config.value = { ...CONFIG.value }
    loading.value = false;
}
defineExpose({ show })

const test = (type) => {
    if (type === 'qb') {
        qbTestLoading.value = true
        api.post('api/test', {
            test_type: 'qb',
            host: config.value.qb_host,
            username: config.value.qb_username,
            password: config.value.qb_password
        }).then(res => {
            ElMessage.success(res.message)
        }).finally(() => {
            qbTestLoading.value = false
        })
        return
    }
    if (type === 'Rclone') {
        RcloneTestLoading.value = true
        api.post('api/test', {
            test_type: 'Rclone',
            host: config.value.rclone_host,
            username: config.value.rclone_username,
            password: config.value.rclone_password
        }).then(res => {
            ElMessage.success(res.message)
        }).finally(() => {
            RcloneTestLoading.value = false
        })
        return
    }
}
</script>