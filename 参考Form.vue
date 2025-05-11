<template>
  <div class="page-wrapper">
    <!-- 固定的顶部区域 -->
    <div class="page-header">
      <div class="header-content">
        <!-- 标题区域 -->
        <!-- <div class="title-section">
          <div class="title">
            <span class="icon">✏️</span>
            笔记数据监测
            <el-tooltip content="查看帮助指南" placement="top">
              <a href="https://jfsq6znqku.feishu.cn/docx/EE1Sdit4VoBMVXxmL4TcZAEDn2P" 
                 target="_blank" 
                 class="help-link">
                <el-icon><question-filled /></el-icon>
              </a>
            </el-tooltip>
          </div>
        </div> -->

        <!-- 步骤条 -->
        <div class="steps">
          <div class="step" :class="{ active: activeStep >= 0, completed: activeStep > 0 }">
            <div class="step-content">
              <div class="step-number">
                <template v-if="activeStep > 0">
                  <el-icon class="check-icon">
                    <check />
                  </el-icon>
                </template>
                <template v-else>1</template>
              </div>
              <div class="step-text">基础配置</div>
            </div>
            <div class="step-line"></div>
          </div>
          <div class="step" :class="{ active: activeStep >= 1 }">
            <div class="step-content">
              <div class="step-number">2</div>
              <div class="step-text">数据监测</div>
            </div>
            <div class="step-line"></div>
          </div>
          <div class="step" :class="{ active: activeStep >= 2 }">
            <div class="step-content">
              <div class="step-number">3</div>
              <div class="step-text">完成</div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 使用 Swiper 替换原有内容区域 -->
    <div class="page-content">
      <div class="content-container">
        <swiper :modules="[Navigation]" :allow-touch-move="false" @swiper="onSwiper" class="form-swiper">
          <!-- 第一步：配置授权码 -->
          <swiper-slide>
            <div class="step-content">
              <el-form ref="form" class="form" label-position="top">
                <el-form-item label="插件授权码" required>
                  <div class="input-tip">请填写卡片下发的 ⌈插件授权码⌋ ~</div>
                  <el-input v-model="token" type="password" placeholder="请输入插件授权码" />
                </el-form-item>

                <el-form-item label="多维表格授权码" required>
                  <div class="input-tip">
                    请填写 ⌈多维表格授权码⌋ ~
                    <el-tooltip content="点击查看获取指南" placement="top">
                      <a href="https://jfsq6znqku.feishu.cn/wiki/E34hw9OzWiMKQ6kWJXWcRkVGnwd" target="_blank"
                        class="help-link">
                        <el-icon><question-filled /></el-icon>
                      </a>
                    </el-tooltip>
                  </div>
                  <el-input v-model="baseToken" placeholder="请输入多维表格授权码" />
                </el-form-item>
              </el-form>
            </div>
          </swiper-slide>

          <!-- 第二步：运行插件 -->
          <swiper-slide>
            <div class="step-content">
              <el-form ref="form" class="form" label-position="top">
                <el-form-item label="链接" required>
                  <el-select v-model="linkFieldId" placeholder="请选择笔记链接对应的字段" class="full-width">
                    <el-option v-for="meta in mainFieldListSeView" :key="meta.id" :label="meta.name" :value="meta.id" />
                  </el-select>
                </el-form-item>

                <div class="settings">
                  <div class="setting-item">
                    <div class="setting-header">
                      <div class="setting-label">
                        <span>图片/视频获取</span>
                        <el-tooltip content="获取图片/视频可能耗时较长，请耐心等待" placement="top">
                          <el-icon class="help-icon"><question-filled /></el-icon>
                        </el-tooltip>
                      </div>
                    </div>
                    <div class="setting-options">
                      <div class="tag-switch">
                        <el-tag :class="{ active: !isFetchFile }" @click="handleMediaFetchChange('none')">
                          不获取
                        </el-tag>
                        <el-tag :class="{ active: isFetchFile && isOnlyFetchSingleFile }"
                          @click="handleMediaFetchChange('single')">
                          仅获取首图
                        </el-tag>
                        <el-tag :class="{ active: isFetchFile && !isOnlyFetchSingleFile }"
                          @click="handleMediaFetchChange('all')">
                          获取全部
                        </el-tag>
                      </div>
                    </div>
                  </div>

                  <div class="setting-item">
                    <div class="setting-label">
                      <span>选取模式</span>
                      <el-tooltip placement="top">
                        <template #content>
                          <div class="tooltip-content">
                            <p><strong>当前视图模式：</strong>将自动获取当前视图下的所有笔记数据</p>
                            <p><strong>选项弹窗模式：</strong>可以手动选择需要获取数据的笔记</p>
                          </div>
                        </template>
                        <el-icon class="help-icon"><question-filled /></el-icon>
                      </el-tooltip>
                    </div>
                    <div class="setting-control">
                      <div class="tag-switch">
                        <el-tag :class="{ active: !fetchRecordMode }" @click="fetchRecordMode = false">当前视图</el-tag>
                        <el-tag :class="{ active: fetchRecordMode }" @click="fetchRecordMode = true">选项弹窗</el-tag>
                      </div>
                    </div>
                  </div>
                </div>
              </el-form>
            </div>
          </swiper-slide>

          <!-- 第三步：完成 -->
          <swiper-slide>
            <div class="form loading-step">
              <div class="loading-content">
                <div id="loadingChart" class="loading-chart"></div>
                <div class="loading-text">
                  <template v-if="showResult">
                    处理完成，{{ successCount }}条成功，{{ failedCount }}条失败
                    <div class="time-info">总耗时：{{ totalTime }}秒</div>
                  </template>
                  <template v-else>
                    已处理 {{ processedCount }} 条笔记，剩余 {{ totalCount - processedCount }} 条
                  </template>
                </div>
              </div>
            </div>
          </swiper-slide>
        </swiper>
      </div>
    </div>

    <!-- 固定的底部区域 -->
    <div class="page-footer">
      <div class="footer-content">
        <el-tooltip content="刷新插件" placement="top">
          <el-button type="text" @click="refreshPlugin" class="refresh-button">
            <el-icon>
              <refresh />
            </el-icon>
          </el-button>
        </el-tooltip>
        <div class="action-buttons">
          <template v-if="activeStep === 0">
            <el-button type="primary" @click="activeStep = 1" :disabled="!isAuthReady">下一步</el-button>
          </template>
          <template v-else>
            <el-button @click="handlePrevStep">上一步</el-button>
            <el-button type="primary" v-loading="isWritingData" @click="writeData" :disabled="!issubmitAbled">
              获取数据
            </el-button>
          </template>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { bitable, FieldType } from '@lark-base-open/js-sdk';
import { useI18n } from 'vue-i18n';
import { ref, onMounted, computed, isShallow, h, nextTick, onUnmounted, watch } from 'vue';
import * as echarts from 'echarts'
import 'echarts-liquidfill'
import { Swiper, SwiperSlide } from 'swiper/vue';
import { Navigation } from 'swiper';
import 'swiper/css';

import { QuestionFilled, EditPen, Check, Refresh } from '@element-plus/icons-vue'
import { completeMappedFieldIdsValue, cleanNoteLink } from '../utils/dataUtils.js'
import { handleError, showErrorToast, handleErrorTip } from '../exceptions/errorHandling.js'
import { BATCH_FETCH_XHS_DATA_URL, ADD_TIMER_URL, MAX_RECORD_NUMS_PER_TIME_FILE_MULT, MAX_RECORD_NUMS_PER_TIME_FILE_SINGLE, MAX_RECORD_NUMS_PER_TIME_COMMON, FETCH_XHS_UPLOADER_DATA_URL } from '../constants.js'
import { requestXHSDataWhenFilled } from '../services/dataService.js'
import { addXHSTimer } from '../services/timerService.js'
// -- 可更改区域
// TODO: 可替换为相应的后端服务基地址，注意末尾没有斜杠
// const baseUrl = ref('https://feishu-xhs-assistant-none-cookie-wuyi.replit.app')  // replit
// const baseUrl = ref('https://xhs-a-fe-cookie-cicfcamgvt.cn-hangzhou.fcapp.run')  // 杭州



// -- 数据区域
const timerForm = ref({
  timer_hour: 0,
  timer_minute: 0,
  timer_tableId: ''
});
const { t } = useI18n();
const mainFieldListSeView = ref()  // 当前视图的数据表字段列表
const viewListSeTable = ref() // 当前数据表的视图列表
const historyFieldListSeView = ref([])
const linkFieldId = ref('')  // 链接字段Id
const versionDesc = ref("")
let historyTable
let historyTableId
const fieldsToMap = ref([
  {
    "label": "title"
  },
  {
    "label": "content"
  },
  {
    "label": "tags"
  },
  {
    "label": "type"
  },
  {
    "label": "releaseTime"
  },
  {
    "label": "lastUpdateTime"
  },
  // {
  //   "label": "viewCount"
  // },
  {
    "label": "collectionCount"
  },
  {
    "label": "likeCount"
  },
  {
    "label": "shareCount"
  },
  {
    "label": "commentCount"
  },
  {
    "label": "totalInterCount"
  },
  {
    "label": "ipLocation"
  },
  {
    "label": "fetchDataTime"
  },
  {
    "label": "errorTip"
  },
  {
    "label": "imageList"
  },
  {
    "label": "uploader"
  },
  {
    "label": "userDesc"
  },
  {
    "label": "userFansCount"
  },
  {
    "label": "userFollowCount"
  },
  {
    "label": "userGender"
  },
  {
    "label": "userHomeLink"
  },
  {
    "label": "userIpLocation"
  },
  {
    "label": "userLikeAndCollectCount"
  },
  {
    "label": "useeRedID"
  },
  {
    "label": "userUserId"
  },
])   //  可以创建的字段
const checkedFieldsToMap = ref([
  'title',
  'content',
  'tags',
  'type',
  'releaseTime',
  'lastUpdateTime',
  // 'viewCount',
  'likeCount',
  'collectionCount',
  'shareCount',
  'commentCount',
  'fetchDataTime',
  'ipLocation',
  'errorTip',
  'totalInterCount',
  'imageList',
  'uploader',
  'userDesc',
  'userFansCount',
  'userFollowCount',
  'userGender',
  'userHomeLink',
  'userIpLocation',
  'userLikeAndCollectCount',
  'userRedID',
  'userUserId'

])   // 默认的to-map的字段

const toCalcInterCount = ref(['likeCount', 'collectionCount', 'shareCount', 'commentCount'])  // 实际用于计算"总交互量"的字段
const allToCalcInterCount = ref({})  // 可用于计算"总交互量"的字段
const token = ref('')
const baseToken = ref('')
const baseIdRequest = ref('')
const tableIdRequest = ref('')
const viewIdForTimer = ref('')
const errorCount = ref(0)

const issubmitAbled = computed(() => {
  if (!isDetailMode.value)
    return linkFieldId.value && checkedFieldsToMap.value.length && token.value.length && baseToken.value.length
  else
    return linkFieldId.value && checkedFieldsToMap.value.length && token.value.length && baseToken.value.length

})  // 是否允许提交，及必选字段是否都填写

const mappedFieldIds = ref({
  "main": {},  // 主表
  "history": {}  // 历史记录 表
})

// -- 配置区域
const isForcedEnd = ref(false)
const isIndeterminateToMap = ref(true)
const isDetailMode = ref(true)
const isShowReward = ref(false)
const isFetchFile = ref(false)
const isOnlyFetchSingleFile = ref(true)
const fetchRecordMode = ref(Boolean(localStorage.getItem('fetchRecordMode')))
const isTimerCardVisible = ref(false)  // 是否展示定时任卡片
const isSetTimer = ref(false)  // 是否设置过定时任务
const isWritingData = ref(false)
const checkAllToMap = ref(false)

const activeStep = ref(0); // 当前步骤

// 在 data 区域添加这些变量
const loadingChartInstance = ref(null)
const processedCount = ref(0)
const totalCount = ref(0)
const showResult = ref(false)
const successCount = ref(0)
const failedCount = ref(0)
const startTime = ref(0)  // 添加开始时间
const totalTime = ref(0)  // 添加总耗时

// 在 data 区域添加计算属性
const isAuthReady = computed(() => {
  return token.value && baseToken.value
})

// 添加 Swiper 相关逻辑
const swiperInstance = ref(null);

const onSwiper = (swiper) => {
  swiperInstance.value = swiper;
};

// 修改步骤切换逻辑
watch(activeStep, (newStep) => {
  if (swiperInstance.value) {
    swiperInstance.value.slideTo(newStep);
  }

  // 更新步骤条状态
  const step2 = document.querySelectorAll('.step')[1]
  if (newStep === 2) {
    // 确保环节二的状态为已完成
    step2.classList.add('completed')
    step2.querySelector('.step-number').innerHTML = '<el-icon class="check-icon"><check /></el-icon>'
  } else if (newStep === 1) {
    // 返回到环节二时，恢复数字状态
    step2.classList.remove('completed')
    step2.querySelector('.step-number').innerText = '2'
  }
});

// -- 核心算法区域
// --001== 写入数据
const writeData = async () => {
  try {
    // 重置所有状态
    processedCount.value = 0
    totalCount.value = 0
    showResult.value = false
    successCount.value = 0
    failedCount.value = 0
    isShowReward.value = false
    errorCount.value = 0

    if (isWritingData.value) {
      isForcedEnd.value = true
    }

    isWritingData.value = true



    // 加载bitable实例
    const { tableId, viewId, baseId } = await bitable.base.getSelection();
    baseIdRequest.value = baseId
    tableIdRequest.value = tableId
    const table = await bitable.base.getActiveTable();
    const view = await table.getViewById(viewId);

    // 错误判断：应当在主表中进行
    if (table.id == historyTable.id) {
      bitable.ui.showToast({
        toastType: 'warning',
        message: t('notAllowedInHistoryTable')
      })
      isWritingData.value = false
      return
    }

    // 获取字段数据Ids object类型，匹配已有的字段，创建缺少的字段
    // @status {mappedFieldIds, isWritingData}  表示是命令性质的方法，改变mappedFieldIds对象的状态
    console.log(351, isFetchFile.value)
    let checkedFields = JSON.parse(JSON.stringify(checkedFieldsToMap.value))
    if (!isFetchFile.value) {
      console.log(1111121321542315)
      checkedFields = checkedFields.filter((item) => item != 'imageList')
    }

    let res = await completeMappedFieldIdsValue(table, historyTable, mainFieldListSeView.value, checkedFields, historyFieldListSeView.value, t)
    isWritingData.value = res.isWritingData
    if (res.isReturn) {
      await bitable.ui.showToast({
        toastType: 'error',
        message: res.errMsg
      })
      isWritingData.value = false
      return
    } else {
      mappedFieldIds.value.main = res.main
      mappedFieldIds.value.history = res.history
    }

    console.log(353, mappedFieldIds.value)

    // 在获取字段映射后添加类型校验
    try {
      await validateFieldTypes(table, mappedFieldIds.value.main);
    } catch (error) {
      isWritingData.value = false
      return // 直接返回，因为validateFieldTypes已经显示了错误提示
    }

    console.log(222222)
    let RecordList = []
    if (!fetchRecordMode.value) {
      // 当前视图模式
      RecordList = await view.getVisibleRecordIdList()
    } else {
      // 弹窗选择模式
      RecordList = await bitable.ui.selectRecordIdList(tableId, viewId);

      // 如果没有选择任何记录
      if (RecordList.length === 0) {
        isWritingData.value = false
        await bitable.ui.showToast({
          toastType: 'warning',
          message: '未选择任何记录'
        })
        return
      }
    }




    console.log("RecordList", RecordList)
    localStorage.setItem('token', token.value)   // string 类型
    localStorage.setItem('baseToken', baseToken.value)   // string 类型
    localStorage.setItem('isDetailMode', isDetailMode.value)   // string 类型
    localStorage.setItem('isFetchFile', isFetchFile.value)   // string 类型
    localStorage.setItem('isOnlyFetchSingleFile', isOnlyFetchSingleFile.value)   // string 类型
    localStorage.setItem('fetchRecordMode', fetchRecordMode.value)   // string 类型


    res = await handleError(linkFieldId.value)
    isWritingData.value = res.isWritingData
    console.log(382, res)
    if (res.isReturn) {
      showErrorToast(res.errMsg)
      isWritingData.value = false
      return
    }

    const data = {
      "base_id": baseIdRequest.value,
      "table_id": tableIdRequest.value,
      "history_table_id": historyTable.id,
      "view_id": viewId,
      "is_fetch_file": isFetchFile.value,
      "base_token": baseToken.value,
      "username": "123213wewe",
      "day_hour": 11,
      "day_minute": 11,
      "is_only_fetch_single_file": isOnlyFetchSingleFile.value,
    }

    const linkField = await table.getFieldById(linkFieldId.value)
    let recordsToHandle = []  // 需要从后端请求数据
    let errorTipFieldIdofMain = mappedFieldIds.value["main"]["errorTip"]
    const MAX_RECORD_NUMS_THIS_TIME = isFetchFile.value ? (isOnlyFetchSingleFile.value ? MAX_RECORD_NUMS_PER_TIME_FILE_SINGLE : MAX_RECORD_NUMS_PER_TIME_FILE_MULT) : MAX_RECORD_NUMS_PER_TIME_COMMON

    totalCount.value = RecordList.length
    startTime.value = Date.now()  // 记录开始时间

    // 在开始处理数据前，切换到第三步
    activeStep.value = 2;

    // 初始化图表
    await nextTick()
    await initLoadingChart()

    // 添加记录更新监听器
    const processedRecords = new Set(); // 用于追踪已处理的记录
    const onRecordModify = (event) => {
      if (!processedRecords.has(event.data.recordId)) {
        processedRecords.add(event.data.recordId);
        processedCount.value = Math.min(processedRecords.size, totalCount.value);
        updateLoadingChart();
      }
    }

    // 订阅记录修改事件
    const unsubscribe = table.onRecordModify(onRecordModify)

    try {
      for (let recordId of RecordList) {
        // 强制退出动作
        if (isForcedEnd.value) {
          isWritingData.value = false
          return
        }

        // 对笔记链接值进行清洗
        let res = await cleanNoteLink(recordId, linkField, t)
        console.log(423, res)
        if (res.isReturn) {
          isWritingData.value = false
          await table.setRecord(recordId, {
            fields: {
              [errorTipFieldIdofMain]: String(res.errMsg)
            }
          })
          return
        }
        if (res.isError) {
          if (res.isEmptyLink) {
            // 空链接视为处理成功,将其加入processedRecords
            processedRecords.add(recordId);
            processedCount.value = Math.min(processedRecords.size, totalCount.value);
            updateLoadingChart();
            continue
          }

          await table.setRecord(recordId, {
            fields: {
              [errorTipFieldIdofMain]: String(res.errMsg)
            }
          })
          continue
        }


        recordsToHandle.push({
          "record_id": recordId,
          "note_link": res.noteLink
        })
      

        // - recordsToHandle MAX_RECORD_NUMS_PER_TIME 个一组，坐满发车（请求一次后端）
        if (recordsToHandle.length == MAX_RECORD_NUMS_THIS_TIME) {
          data["records_need_update"] = recordsToHandle
          const request_res = await requestXHSDataWhenFilled(BATCH_FETCH_XHS_DATA_URL, data, token.value, mappedFieldIds.value)
          if (request_res.isReturn) {
            isWritingData.value = false
            isShowReward.value = true
            await bitable.ui.showToast({
              toastType: 'error',
              message: request_res.errMsg
            })
            return
          }




          recordsToHandle = []  // - 人下车

        }
      }


        // - 最后一辆车，有人就发车，没人就不发车
      if (recordsToHandle.length != 0) {
        data["records_need_update"] = recordsToHandle
        const request_res = await requestXHSDataWhenFilled(BATCH_FETCH_XHS_DATA_URL, data, token.value, mappedFieldIds.value)
        if (request_res.isReturn) {
          isWritingData.value = false
          isShowReward.value = true
          await bitable.ui.showToast({
            toastType: 'error',
            message: request_res.errMsg
          })
          return
        }


        recordsToHandle = []  // - 人下车

      }

      isWritingData.value = false
      isShowReward.value = true

      // 如果有未处理的记录，显示相应提示
      if (processedRecords.size < totalCount.value) {
        showResult.value = true
        successCount.value = processedRecords.size
        failedCount.value = totalCount.value - processedRecords.size
        processedCount.value = totalCount.value // 确保进度条显示100%
        totalTime.value = Math.round((Date.now() - startTime.value) / 1000)  // 计算总耗时(秒)
        updateLoadingChart()
        await bitable.ui.showToast({
          toastType: 'warning',
          message: `有 ${totalCount.value - processedRecords.size} 条记录未能成功处理`
        });
      } else {
        showResult.value = true
        successCount.value = totalCount.value
        failedCount.value = errorCount.value
        processedCount.value = totalCount.value // 确保进度条显示100%
        totalTime.value = Math.round((Date.now() - startTime.value) / 1000)  // 计算总耗时(秒)
        updateLoadingChart()
        await bitable.ui.showToast({
          toastType: 'success',
          message: `${t('finishTip')} ${errorCount.value}`
        });
      }

      // 在最后取消订阅
      unsubscribe()
    } catch (error) {
      // 发生错误时也要取消订阅
      unsubscribe()
      throw error
    }
  } catch (error) {
    isWritingData.value = false
    isShowReward.value = true

    // 显示具体的错误信息
    await bitable.ui.showToast({
      toastType: 'error',
      message: error.message || '表格异常错误'
    })
  }
}

// 修改 validateFieldTypes 函数
const validateFieldTypes = async (table, mappedFieldIds) => {
  const fieldTypeNames = {
    [FieldType.Text]: '文本',
    [FieldType.Number]: '数字',
    [FieldType.DateTime]: '日期时间',
    [FieldType.URL]: '超链接'
  };

  const fieldTypes = {
    title: FieldType.Text,
    content: FieldType.Text,
    tags: FieldType.Text,
    type: FieldType.Text,
    releaseTime: FieldType.DateTime,
    lastUpdateTime: FieldType.DateTime,
    collectionCount: FieldType.Number,
    likeCount: FieldType.Number,
    shareCount: FieldType.Number,
    commentCount: FieldType.Number,
    totalInterCount: FieldType.Number,
    ipLocation: FieldType.Text,
    fetchDataTime: FieldType.DateTime,
    errorTip: FieldType.Text,
    imageList: FieldType.Attachment,
    uploader: FieldType.Text,
    userDesc: FieldType.Text,
    userFansCount: FieldType.Number,
    userFollowCount: FieldType.Number,
    userGender: FieldType.Text,
    userHomeLink: [FieldType.Text, FieldType.Url], // 允许文本或超链接类型
    userIpLocation: FieldType.Text,
    userLikeAndCollectCount: FieldType.Number,
    userRedID: FieldType.Text,
    userUserId: FieldType.Text
  };

  // 先获取所有字段的信息
  const allFields = await table.getFieldMetaList();
  const fieldMap = new Map(allFields.map(field => [field.id, field]));

  for (const [fieldName, fieldId] of Object.entries(mappedFieldIds)) {
    const field = fieldMap.get(fieldId);
    if (!field) {
      continue; // 跳过未找到的字段
    }

    const expectedType = fieldTypes[fieldName];
    if (!expectedType) {
      continue; // 跳过未定义类型要求的字段
    }

    // 检查字段类型是否符合要求（支持多个允许的类型）
    const allowedTypes = Array.isArray(expectedType) ? expectedType : [expectedType];
    if (!allowedTypes.includes(field.type)) {
      const typeNames = allowedTypes.map(type => fieldTypeNames[type] || type).join(' 或 ');
      const errorMessage = `字段 "${field.name}" 的类型不匹配，\n需要类型: ${typeNames}。\n\n请修改字段类型后，点击左下角刷新按钮重试`;

      await bitable.ui.showToast({
        toastType: 'error',
        message: errorMessage
      });

      throw new Error(errorMessage);
    }
  }
};

// -- 辅助算法区域





/**
 * --010== 查询式，获取 recordFields
 * @param {object} totalNoteInfo 笔记数据
 * @param {object} mappedFieldIds 字段链接
 */
const getRecordFields = (totalNoteInfo, mappedFieldIds, noteLink = 0) => {
  let recordFields = {}
  let key = ''
  let value = ''
  let fetchDataTimeValue = Date.now()
  console.log(mappedFieldIds)
  let checkedFields = JSON.parse(JSON.stringify(checkedFieldsToMap.value))

  checkedFields = checkedFields.filter(item => item != 'errorTip')
  for (let field of checkedFields) {

    key = mappedFieldIds[field]

    if (field === 'fetchDataTime')
      value = fetchDataTimeValue
    else if (field === 'totalInterCount')
      value = toCalcInterCount.value.reduce((total, key) => total + totalNoteInfo.basicInfo[key], 0);
    else
      value = totalNoteInfo.basicInfo[field]


    recordFields[key] = value
  }

  // 打个补丁，链接字段
  if (noteLink)
    recordFields[mappedFieldIds['link']] = noteLink


  return recordFields
}

/**
 * * --011== 查询获取定时任务设置
 * @query {}
 */

const openMessageBox = async () => {

  if (!linkFieldId.value.length) {
    bitable.ui.showToast({
      toastType: 'warning',
      message: "请选择笔记链接对应的字段"
    })
    return
  }
  if (!token.value.length) {
    bitable.ui.showToast({
      toastType: 'warning',
      message: "请输入插件授权码"
    })
    return
  }
  if (!baseToken.value.length) {
    bitable.ui.showToast({
      toastType: 'warning',
      message: "请输入多维表格授权码"
    })
    return
  }


  // 加载bitable实例
  const { tableId, viewId, baseId } = await bitable.base.getSelection();
  baseIdRequest.value = baseId
  tableIdRequest.value = tableId
  const table = await bitable.base.getActiveTable();

  // 错误判断：应当在主表中进行
  if (table.id == historyTable.id) {
    bitable.ui.showToast({
      toastType: 'warning',
      message: t('notAllowedInHistoryTable')
    })
    isWritingData.value = false
    return
  }

  const RecordList = await bitable.ui.selectRecordIdList(tableId, viewId);
  if (RecordList.length > 200) {
    bitable.ui.showToast({
      toastType: 'warning',
      message: "当前版本支持定时运行最多200条笔记。"
    })
    return
  }
  const records_need_update = []
  for (let recordId of RecordList) {
    // 强制退出动作
    if (isForcedEnd.value) {
      isWritingData.value = false
      return
    }

    // 对笔记链接值进行清洗
    const linkField = await table.getFieldById(linkFieldId.value)
    let res = await cleanNoteLink(recordId, linkField, t)
    if (res.isReturn) {
      isWritingData.value = false
      return
    }
    if (res.isError)
      continue

    records_need_update.push({
      "record_id": recordId,
      "note_link": res.noteLink,
      "short_url": res.noteLink  // 添加short_url参数，与note_link相同
    })
  }







  ElMessageBox({
    title: '⌛ 设置定时任务 ',
    message: () =>
      h('div', { style: 'padding: 20px;' }, [
        h('div', { style: 'margin-bottom: 16px;' }, [
          h('label', { style: 'display: block;  margin-bottom: 5px;' }, '定时触发小时：'),
          h('input', {
            type: 'number',
            min: '0',
            max: '23',
            value: timerForm.value.timer_hour,
            placeholder: '请输入小时',
            style: 'width: calc(100% - 20px); padding: 8px; margin-top: 5px; border-radius: 4px;border: 1px solid #b6b8c1;',
            onInput: (event) => {
              timerForm.value.timer_hour = (event.target).value;
            }
          })
        ]),
        h('div', { style: 'margin-bottom: 16px;' }, [
          h('label', { style: 'display: block;  margin-bottom: 5px;' }, '定时触发分钟：'),
          h('input', {
            type: 'number',
            min: '0',
            max: '59',
            placeholder: '请输入分钟',
            value: timerForm.value.timer_minute,
            style: 'width: calc(100% - 20px); padding: 8px; margin-top: 5px; border-radius: 4px;border: 1px solid #b6b8c1;',
            onInput: (event) => {
              timerForm.value.timer_minute = (event.target).value;
            }
          })
        ])
      ]),
    dangerouslyUseHTMLString: false,
    showCancelButton: true,
    confirmButtonText: '确定',
    cancelButtonText: '取消',
    beforeClose: async (action, instance, done) => {
      if (action === 'confirm') {
        const { timer_hour, timer_minute } = timerForm.value;
        if (isNaN(timer_hour) || isNaN(timer_minute)) {
          ElMessage({
            message: '小时和分钟必须是数字',
            type: 'error',
            plain: true,
          });
          return;
        }
        if ((!timer_hour || !timer_minute) && timer_hour != 0 && timer_minute != 0) {
          ElMessage({
            message: '请填写所有字段',
            type: 'error',
            plain: true,
          });
          return;
        }

        if (timer_hour < 0 || timer_hour > 23 || timer_minute < 0 || timer_minute > 59) {
          ElMessage({
            message: '小时和分钟的值必须在有效范围内',
            type: 'error',
            plain: true,
          });
          return;
        }

        console.log('定时任务设置:', { timer_hour, timer_minute });
        isSetTimer.value = true
        const { tableId, viewId, baseId } = await bitable.base.getSelection();
        timerForm.value.timer_tableId = tableId
        localStorage.setItem('xhs-a01-isSetTimer-v2', isSetTimer.value)   // string 类型
        localStorage.setItem('xhs-a01-timer-hour-v2', timerForm.value.timer_hour)   // string 类型
        localStorage.setItem('xhs-a01-timer-minute-v2', timerForm.value.timer_minute)   // string 类型
        localStorage.setItem('xhs-a01-tableId-v2', timerForm.value.timer_tableId)   // string 类型

      
        // 确保至少有一条记录
        if (records_need_update.length === 0) {
          ElMessage({
            message: '未选择有效的记录，无法设置定时任务',
            type: 'error',
            plain: true
          });
          return;
        }

        // 构建定时任务请求数据
        const timerData = {
          "base_id": baseId,
          "table_id": tableId,
          "history_table_id": historyTable.id,
          "view_id": viewId,
          "is_fetch_file": false,
          "base_token": baseToken.value,
          "username": "feishu",
          "day_hour": timerForm.value.timer_hour,
          "day_minute": timerForm.value.timer_minute,
          "is_only_fetch_single_file": false,
          "record_id": records_need_update[0].record_id,
          "short_url": records_need_update[0].short_url,
        }

        console.log("定时任务请求数据:", timerData)

        const res = await addXHSTimer(ADD_TIMER_URL, timerData, token.value)
        if (res.isError) {
          bitable.ui.showToast({
            toastType: 'error',
            message: res.errMsg
          })
        }
        bitable.ui.showToast({
          toastType: 'success',
          message: "定时任务设置成功 ~"
        })
        done();
      } else {
        done();
      }
    }
  });
};


// Map==全选事件
const handlecheckAllToMapChange = (val) => {
  const data = JSON.parse(JSON.stringify(fieldsToMap.value))
  if (val) {
    checkedFieldsToMap.value = []
    for (const item of data)
      checkedFieldsToMap.value.push(item.label);


  } else {
    checkedFieldsToMap.value = []
  }
  isIndeterminateToMap.value = false
}
// Map==字段选择事件
const handleCheckedFieldsToMapChange = (value) => {
  const checkedCount = value.length
  checkAllToMap.value = checkedCount === fieldsToMap.value.length
  isIndeterminateToMap.value = checkedCount > 0 && checkedCount < fieldsToMap.value.length
}

// 添加处理媒体获取模式变更的方法
const handleMediaFetchChange = (mode) => {
  switch (mode) {
    case 'none':
      isFetchFile.value = false
      isOnlyFetchSingleFile.value = false
      break
    case 'single':
      isFetchFile.value = true
      isOnlyFetchSingleFile.value = true
      break
    case 'all':
      isFetchFile.value = true
      isOnlyFetchSingleFile.value = false
      break
  }
  // 保存设置到本地存储
  localStorage.setItem('isFetchFile', isFetchFile.value)
  localStorage.setItem('isOnlyFetchSingleFile', isOnlyFetchSingleFile.value)
}

onMounted(async () => {
  // 初始化勾选字段
  // const language = await bitable.bridge.getLanguage();

  // 获取字段列表 -- start
  const selection = await bitable.base.getSelection()
  const table = await bitable.base.getTableById(selection.tableId)
  const view = await table.getViewById(selection.viewId)
  mainFieldListSeView.value = await view.getFieldMetaList()
  // 获取字段列表 -- end

  viewListSeTable.value = await table.getViewMetaList()
  // 历史记录表
  try {
    historyTable = await bitable.base.getTableByName(t('history.tableName'));



  } catch (error) {
    const { tableId, index } = await bitable.base.addTable({
      name: t('history.tableName')
    })
    historyTable = await bitable.base.getTableById(tableId)

  }

  historyFieldListSeView.value = await historyTable.getFieldMetaList()


  // 初始化可参与计算 "总交互量" 的对象数组，以"Count"结尾的
  allToCalcInterCount.value = fieldsToMap.value
    .filter(item => item.label.endsWith('Count'));

  if (localStorage.getItem('isDetailMode') !== null) {  // string 类型
    isDetailMode.value = Boolean(localStorage.getItem('isDetailMode'))
  }
  if (localStorage.getItem('token') !== null) {  // string 类型
    token.value = localStorage.getItem('token')
  }
  if (localStorage.getItem('baseToken') !== null) {  // string 类型
    baseToken.value = localStorage.getItem('baseToken')
  }
  if (localStorage.getItem('xhs-a01-isSetTimer-v2') !== null) {  // string 类型
    isSetTimer.value = localStorage.getItem('xhs-a01-isSetTimer-v2')
  }
  if (localStorage.getItem('xhs-a01-timer-hour-v2') !== null) {  // string 类型
    timerForm.value.timer_hour = localStorage.getItem('xhs-a01-timer-hour-v2')
  }
  if (localStorage.getItem('xhs-a01-timer-minute-v2') !== null) {  // string 类型
    timerForm.value.timer_minute = localStorage.getItem('xhs-a01-timer-minute-v2')
  }
  if (localStorage.getItem('xhs-a01-tableId-v2') !== null) {  // string 类型
    timerForm.value.timer_tableId = localStorage.getItem('xhs-a01-tableId-v2')
  }



  // axios 请求 https://jfsq6znqku.feishu.cn/docx/H6kedtvxPoohWixQE5ocytPCn6f，获取 meta 标签



  // 如果两个授权码都已存在，自动跳转到第二步
  if (token.value && baseToken.value) {
    activeStep.value = 1
  }
});

// 添加以下方法:
const initLoadingChart = async () => {
  await nextTick()
  const chartDom = document.getElementById('loadingChart')
  if (!chartDom) return

  loadingChartInstance.value = echarts.init(chartDom)

  const option = {
    backgroundColor: 'transparent',
    series: [{
      type: 'liquidFill',
      data: [0],
      radius: '80%',
      amplitude: 20,
      color: ['#409eff'],
      backgroundStyle: {
        color: '#f5f7fa'
      },
      outline: {
        show: false
      },
      label: {
        show: true,
        fontSize: 28,
        fontFamily: 'Arial',
        color: '#409eff',
        insideColor: '#fff',
        formatter: () => {
          return `${Math.round(processedCount.value / totalCount.value * 100)}%`
        }
      }
    }]
  }

  loadingChartInstance.value.setOption(option)
}

const updateLoadingChart = () => {
  if (!loadingChartInstance.value) return

  // 确保分母不为0
  const total = totalCount.value || 1
  const progress = Math.min(processedCount.value / total, 1)

  loadingChartInstance.value.setOption({
    series: [{
      data: [progress],
      label: {
        formatter: () => {
          return `${Math.round(progress * 100)}%`
        }
      }
    }]
  })
}

onUnmounted(() => {
  if (loadingChartInstance.value) {
    loadingChartInstance.value.dispose()
  }
})

// 添加 handlePrevStep 方法
const handlePrevStep = () => {
  if (activeStep.value === 2) {
    // 如果当前是环节三，返回到环节二
    activeStep.value = 1
  } else if (activeStep.value === 1) {
    // 如果当前是环节二，返回到环节一
    activeStep.value = 0
  }
}

// 添加 refreshPlugin 方法
const refreshPlugin = () => {
  window.location.reload()
}

</script>



<style scoped>
/* 页面整体布局 */
.page-wrapper {
  height: 100vh;
  /* 保持94vh，避免出现滚动条 */
  display: flex;
  flex-direction: column;
  background: #f7f8fa;
  overflow: hidden;
  padding: 0;
}

/* 顶部区域 */
.page-header {
  height: 80px;
  /* 保持顶部区域高度不变 */
  background: transparent;
  box-shadow: none;
}

.header-content {
  width: 80vw;
  margin: 0 auto;
  padding: 6px 0 4px;
  height: 100%;
  display: flex;
  flex-direction: column;
}

/* 标题区域 */
.title-section {
  padding: 0;
  /* 移除左右内边距，与内容对齐 */
  margin-bottom: 6px;
}

.title {
  font-size: 15px;
  display: flex;
  align-items: center;
  gap: 6px;
  color: #1f2329;
}

.help-link {
  display: flex;
  align-items: center;
  color: #8f959e;
  transition: color 0.2s;
}

.help-link:hover {
  color: #3370ff;
}

/* 步骤条样式 */
.steps {
  display: flex;
  justify-content: space-between;
  padding: 4px 0 8px;
  /* 移除左右内边距，与内容对齐 */
  position: relative;
  align-items: center;
  flex: 1;
}

.step {
  position: relative;
  flex: 1;
  text-align: center;
  max-width: 120px;
}

.step-content {
  position: relative;
  z-index: 2;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  height: 88%;
}

.step-number {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #f0f2f5;
  color: #909399;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 11px;
  transition: all 0.3s ease;
}

.step-text {
  font-size: 11px;
  color: #909399;
  white-space: nowrap;
}

.step-line {
  position: absolute;
  top: 9px;
  left: 48px;
  right: -50%;
  height: 1px;
  background: #e5e7eb;
  z-index: 1;
}

.step:last-child .step-line {
  display: none;
}

.step.active .step-number {
  background: #3370ff;
  color: white;
}

.step.active .step-text {
  color: #1f2329;
}

.step.completed .step-number {
  background: #f0f2f5;
  color: #909399;
}

.step.completed .step-text {
  color: #1f2329;
}

.check-icon {
  font-size: 11px;
  color: #909399;
}

/* 内容区域 */
.page-content {
  flex: 1;
  padding: 20px 0;
  /* 增加上下内边距 */
  overflow: hidden;
}

.content-container {
  width: 90vw;
  /* 从80vw增加到90vw，使卡片更宽 */
  height: 100%;
  margin: 0 auto;
}

/* 底部区域 */
.page-footer {
  height: 60px;
  /* 保持底部区域高度不变 */
  background: white;
  box-shadow: 0 -1px 2px rgba(0, 0, 0, 0.05);
}

.footer-content {
  width: 80vw;
  margin: 0 auto;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  /* 左右分开 */
}

/* 设置项样式 */
.setting-item {
  padding: 16px 0;
}

.setting-item:last-child {
  border-bottom: none;
}

.setting-header {
  margin-bottom: 12px;
}

.setting-label {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 14px;
  color: #1f2329;
}

.setting-options {
  padding-left: 4px;
  /* 添加左侧缩进 */
}

.tag-switch {
  display: flex;
  gap: 8px;
}

.tag-switch .el-tag {
  cursor: pointer;
  padding: 12px 16px;
  transition: all 0.2s;
  background-color: #f0f2f5;
  border-color: #f0f2f5;
  color: #909399;
  font-size: 13px;
}

.tag-switch .el-tag.active {
  background-color: #3370ff;
  border-color: #3370ff;
  color: white;
}

.tag-switch .el-tag:hover {
  border-color: #3370ff;
}

/* 工具提示样式 */
.tooltip-content {
  font-size: 13px;
  line-height: 1.6;
}

.tooltip-content p {
  margin: 0;
  padding: 4px 0;
}

.tooltip-content strong {
  color: #fff;
  font-weight: 500;
}

/* Element Plus 样式覆盖 */
:deep(.el-form-item__label) {
  font-weight: 500;
  color: #1f2329;
}

:deep(.el-input__inner) {
  border-color: #dcdfe6;
}

:deep(.el-button--primary) {
  background-color: #3370ff;
}

:deep(.el-button--primary:hover) {
  background-color: #528bff;
}

/* 工具类 */
.full-width {
  width: 100%;
}

/* 修改加载容器样式 */
.loading-container {
  display: flex;
  align-items: center;
  justify-content: center;
  background: white;
  height: 100%;
}

.loading-content {
  text-align: center;
  padding: 32px;
}

.loading-chart {
  width: 180px;
  height: 180px;
  margin-bottom: 16px;
}

.loading-text {
  margin-bottom: 8px;
  /* 添加底部间距 */
  font-size: 14px;
  color: #646a73;
}

/* 修改相关样式 */
.form {
  height: 100%;
  width: 80%;
  background: white;
  border-radius: 12px;
  padding: 32px;
  /* 增加内边距 */
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.06);
  display: flex;
  flex-direction: column;
}

/* 表单内容区域 - 允许内部滚动 */


/* 设置项容器 */
.settings {
  margin-top: 24px;
  padding-bottom: 24px;
  /* 添加底部内边距，避免内容贴边 */
}

/* 表单项样式 */
:deep(.el-form-item) {
  margin-bottom: 16px;
}

:deep(.el-form-item:last-child) {
  margin-bottom: 0;
}

.input-tip {
  font-size: 14px;
  color: #646a73;
  margin-bottom: 8px;
  display: flex;
  align-items: center;
  gap: 4px;
}

/* 添加 Swiper 相关样式 */
.form-swiper {
  height: 100%;
}

:deep(.swiper-slide) {
  height: 100%;
  overflow-y: auto;
}

/* 修改内容区域样式 */
.page-content {
  flex: 1;
  padding: 12px 0;
  overflow: hidden;
}

.content-container {
  width: 80vw;
  height: 100%;
  margin: 0 auto;
}

/* 加载步骤样式 */
.loading-step {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  background: white;
  border-radius: 12px;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.06);
}

.loading-content {
  text-align: center;
}

.loading-chart {
  width: 240px;
  height: 240px;
  margin-bottom: 24px;
}

.loading-text {
  font-size: 14px;
  color: #646a73;
}

/* 确保第三步不出现滚动条 */
:deep(.swiper-slide:last-child) {
  overflow: hidden;
}

/* 在样式部分添加以下内容 */
.refresh-button {
  color: #909399;
  transition: color 0.2s;
  margin-right: auto;
  /* 靠左 */
}

.refresh-button:hover {
  color: #3370ff;
}

/* 操作按钮容器 */
.action-buttons {
  display: flex;
  gap: 12px;
  margin-left: auto;
  /* 靠右 */
}

/* 添加结果文字样式 */
.result-text {
  margin-top: 12px;
  font-size: 14px;
  color: #1f2329;
  font-weight: 500;
}

.time-info {
  margin-top: 12px;
  font-size: 13px;
  color: #909399;
}

.setting-label {
  margin-bottom: 12px;
}
</style>
