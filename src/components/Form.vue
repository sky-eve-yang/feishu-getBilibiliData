<template>
  <el-form ref="form" class="form" label-position="left">
    <div style="width: 100%;padding-left: 10px;border-left: 5px solid #2598f8;margin-bottom: 20px;padding-top: 5px;">{{ $t('title') }}</div>
    <el-alert style="margin: 20px 0;color: #606266;" :title="$t('alerts.selectNumberField')" type="info" />

    <el-form-item style="margin-top: 40px;" :label="$t('labels.link')" size="large" required>
      <el-select v-model="linkFieldId" :placeholder="$t('placeholder.link')" style="width: 100%">
        <el-option v-for="meta in fieldListSeView" :key="meta.id" :label="meta.name" :value="meta.id" />
      </el-select>
    </el-form-item>
    
    <!-- 多选框 -->
    <div class="map-fields-checklist">
      <el-checkbox v-model="checkAllToMap" :indeterminate="isIndeterminateToMap" @change="handlecheckAllToMapChange">{{
        $t('selectGroup.selectAll') }}</el-checkbox>
      <el-checkbox-group v-model="checkedFieldsToMap" @change="handleCheckedFieldsToMapChange">
        <el-checkbox v-for="fieldToMap in fieldsToMap" :key="fieldToMap.label" :label="fieldToMap.label">
          {{ $t(`selectGroup.videoInfo.${fieldToMap.label}`) }}
        </el-checkbox>
      </el-checkbox-group>
    </div>
    <div style="margin-top: 20px;" v-if="checkedFieldsToMap.includes('totalInterCount')">
      <el-select
        
        v-model="toCalcInterCount"
        multiple
        :placeholder="$t('placeholder.interCount')"
        style="width: 100%;"
        size="large"
      >
        <el-option
          v-for="item in allToCalcInterCount"
          :key="item.label"
          :label="$t(`selectGroup.videoInfo.${item.label}`)"
          :value="item.label"
        />
      </el-select>
    </div>
    <el-alert style="display: flex;align-items: flex-start;margin: 20px 0;background-color: #e1eaff;color: #606266;" :title="$t('alerts.selectGroupFieldTip')" type="info" show-icon />


    <!-- 选择对应的字段映射 -->

    <!-- 选择刷新模式 -->
    <el-form-item style="margin-top: 30px;" :label="$t('refreshMode.title')" size="large">
      <el-radio-group v-model="refreshMode">
        <el-radio :label="'current'">{{ $t('refreshMode.current') }}</el-radio>
        <el-radio :label="'select'">{{ $t('refreshMode.select') }}</el-radio>
      </el-radio-group>
    </el-form-item>

    <!-- 提交按钮 -->
    <el-button v-loading="isWritingData" @click="writeData" :disabled="!issubmitAbled || isWritingData" color="#3370ff" type="primary" plain size="large">{{ $t('submit') }}</el-button>
  </el-form>

  <!-- 加载进度对话框 -->
  <el-dialog
    v-model="showProgressDialog"
    :title="$t('progressDialog.title')"
    width="400px"
    :close-on-click-modal="false"
    :close-on-press-escape="false"
    :show-close="false"
  >
    <div class="loading-content">
      <div id="loadingChart" class="loading-chart"></div>
      <div class="loading-text">
        <template v-if="showResult">
          处理完成，{{ successCount }}条成功，{{ failedCount }}条失败
          <div class="time-info">总耗时：{{ totalTime }}秒</div>
        </template>
        <template v-else>
          已处理 {{ processedCount }} 条记录，剩余 {{ totalCount - processedCount }} 条
        </template>
      </div>
    </div>
    <template #footer>
      <div class="dialog-footer">
        <el-button v-if="showResult" @click="closeProgressDialog">关闭</el-button>
        <el-button v-else type="danger" @click="cancelProcessing">取消</el-button>
      </div>
    </template>
  </el-dialog>
</template>

<script setup>
import { bitable, FieldType } from '@lark-base-open/js-sdk';
import { useI18n } from 'vue-i18n';
import { ref, onMounted, computed, nextTick, onUnmounted } from 'vue';
import axios from 'axios';
import qs from 'qs';
import * as echarts from 'echarts';
import 'echarts-liquidfill';
import { ElLoading } from 'element-plus';


// -- 数据区域
const { t } = useI18n();
const fieldListSeView = ref([])
const linkFieldId = ref('')  // 链接字段Id

const isWritingData = ref(false)

const checkAllToMap = ref(false)
const isIndeterminateToMap = ref(true)
const fieldsToMap = ref([
  {
    "label": "title"
  },
  {
    "label": "uploader"
  },
  {
    "label": "releaseTime"
  },
  {
    "label": "danmuCount"
  },
  {
    "label": "coinCount"
  },
  {
    "label": "viewCount"
  },
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
    "label": "commentWc"
  },
  {
    "label": "danmuWc"
  },
  {
    "label": "totalInterCount"
  },
  {
    "label": "fetchDataTime"
  }
])   //  可以创建的字段
const checkedFieldsToMap = ref([
  'title',
  'uploader',
  'releaseTime',
  'danmuCount',
  'coinCount',
  'viewCount',
  'collectionCount',
  'likeCount',
  'shareCount',
  'commentCount',
  'fetchDataTime'
])   // 默认的to-map的字段

const refreshMode = ref('current') // 刷新模式：current-当前视图，select-选择记录

const toCalcInterCount = ref(['likeCount','collectionCount', 'shareCount', 'commentCount'])  // 实际用于计算"总交互量"的字段
const allToCalcInterCount = ref({})  // 可用于计算"总交互量"的字段

// 进度相关变量
const loadingChartInstance = ref(null)
const processedCount = ref(0)
const totalCount = ref(0)
const showResult = ref(false)
const successCount = ref(0)
const failedCount = ref(0)
const startTime = ref(0)
const totalTime = ref(0)
const showProgressDialog = ref(false)
const isCancelled = ref(false)

const issubmitAbled = computed(() => {
  return linkFieldId.value && checkedFieldsToMap.value.length
})  // 是否允许提交，及必选字段是否都填写

const mappedFieldIds = ref({})

// 关闭进度对话框
const closeProgressDialog = () => {
  showProgressDialog.value = false
}

// 取消处理
const cancelProcessing = async () => {
  isCancelled.value = true
  await bitable.ui.showToast({
    toastType: 'info',
    message: '正在取消处理...'
  })
}


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
    isCancelled.value = false
    
    // 获取字段数据Ids object类型
    isWritingData.value = true
    
    // 显示加载指示器，表明正在准备
    const loadingInstance = ElLoading.service({
      lock: true,
      text: '准备字段映射中...',
      background: 'rgba(255, 255, 255, 0.7)'
    });
    
    // 先准备所需的字段，不显示进度条
    try {
      const mappedFields = getSelectedFieldsId(fieldListSeView.value, checkedFieldsToMap.value)
      console.log("writeData() >> original mappedFields", mappedFields)
      if (typeof mappedFields == 'string') // 错误处理，提示格式错误 
      {
        loadingInstance.close();
        await bitable.ui.showToast({
          toastType: 'warning',
          message: mappedFields
        })
        isWritingData.value = false
        return
      }

      // 创建缺少的字段
      mappedFieldIds.value = mappedFields
      await createFields()

      console.log("writeData() >> finished mappedFieldIds", mappedFieldIds.value)
      
      // 关闭准备阶段的加载指示器
      loadingInstance.close();
    } catch (error) {
      loadingInstance.close();
      throw error;
    }
    
    // 加载bitable实例
    const { tableId, viewId } = await bitable.base.getSelection();
    const table = await bitable.base.getActiveTable();
    const view = await table.getViewById(viewId);

    let RecordList = [];
    
    // 根据刷新模式获取记录列表
    if (refreshMode.value === 'current') {
      // 获取当前视图的所有记录
      RecordList = await view.getVisibleRecordIdList();
      
      // 在获取记录后设置开始时间
      startTime.value = Date.now();
      
      // 立即显示进度条对话框
      showProgressDialog.value = true;
    } else if (refreshMode.value === 'select') {
      // 交互式选择记录 - 此时不显示进度条
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
      
      // 在选择记录后设置开始时间，以保证计时准确
      startTime.value = Date.now();
      
      // 在选择记录后，再显示进度条对话框
      showProgressDialog.value = true;
    }

    totalCount.value = RecordList.length
    
    // 初始化图表
    await nextTick()
    await initLoadingChart()

    // 添加记录处理计数
    const processedRecords = new Set(); 
    
    // 批量处理的大小，提高性能
    const BATCH_SIZE = 5;
    // 分批处理记录，避免一次处理太多导致性能问题
    for (let batchStart = 0; batchStart < RecordList.length; batchStart += BATCH_SIZE) {
      // 获取当前批次的记录
      const batchRecords = RecordList.slice(batchStart, batchStart + BATCH_SIZE);
      
      // 创建处理当前批次所有记录的Promise数组
      const batchPromises = batchRecords.map(async (recordId) => {
        // 检查是否取消处理
        if (isCancelled.value) {
          return;
        }

        try {
          // 获取视频链接
          console.log("writeData() >> recordId", recordId)
          
          let videoLink = null
          try {
            videoLink = await getCellValueByRFIDS(recordId, linkFieldId.value)
            
            // 如果链接为空或无效，将其视为处理成功，而非失败
            if (!videoLink || typeof videoLink !== 'string' || !videoLink.trim()) {
              console.log("空链接记录处理成功:", recordId)
              // 更新成功计数
              successCount.value++
              processedCount.value++
              processedRecords.add(recordId)
              updateLoadingChart()
              return;
            }
          } catch (error) {
            console.error("获取链接失败:", error)
            failedCount.value++
            processedCount.value++
            updateLoadingChart()
            return;
          }
          console.log("writeData() >> videoLink", videoLink)

          // 获取所有数据
          const totalVideoInfo = await getDataByCheckedFields(videoLink)
          console.log("writeData() >> totalVideoInfo", totalVideoInfo)

          // 准备记录字段数据
          const recordFields = {}
          
          for (let field of checkedFieldsToMap.value) {
            console.log("field", field)
            if (field == 'uploader' || field == 'title') {  // up主字段和标题字段，Text 格式
              recordFields[mappedFieldIds.value[field]] = [{ type: 'text', text: totalVideoInfo.basicInfo[field] }]
            } else if (field.endsWith('Count') && field !== 'totalInterCount') {  // xx量字段，Number格式 注：总互动量字段单独处理
              recordFields[mappedFieldIds.value[field]] = totalVideoInfo.basicInfo[field]
            } else if (field === 'releaseTime') { // 发布时间，datetime格式
              recordFields[mappedFieldIds.value[field]] = totalVideoInfo.basicInfo[field]
            } else if (field === 'fetchDataTime') { // 数据获取时间，datetime格式
              recordFields[mappedFieldIds.value[field]] = Date.now()
            } else if (field === 'totalInterCount') { // 总互动量
              const totalInterCount = toCalcInterCount.value.reduce((total, key) => total + totalVideoInfo.basicInfo[key], 0);
              recordFields[mappedFieldIds.value[field]] = totalInterCount
            }
          }

          // 一次性更新记录的所有字段（不包括附件字段）
          await table.setRecord(recordId, { fields: recordFields })

          // 单独处理附件字段 (commentWc, danmuWc)
          if (checkedFieldsToMap.value.includes('commentWc')) {
            const attachmentField = await table.getFieldById(mappedFieldIds.value['commentWc'])
            await attachmentField.setValue(recordId, totalVideoInfo.commentWc)
          }
          
          if (checkedFieldsToMap.value.includes('danmuWc')) {
            const attachmentField = await table.getFieldById(mappedFieldIds.value['danmuWc'])
            await attachmentField.setValue(recordId, totalVideoInfo.barrageWc)
          }

          // 更新进度
          processedRecords.add(recordId);
          processedCount.value = processedRecords.size;
          updateLoadingChart();
          
          // 更新成功计数
          successCount.value++;
        } catch (error) {
          console.error("处理记录时出错:", error);
          failedCount.value++;
          processedCount.value++;
          updateLoadingChart();
        }
      });
      
      // 等待当前批次的所有记录处理完成
      await Promise.all(batchPromises);
      
      // 检查是否已取消处理
      if (isCancelled.value) {
        isWritingData.value = false
        showProgressDialog.value = false
        showResult.value = true
        totalTime.value = Math.round((Date.now() - startTime.value) / 1000)
        await bitable.ui.showToast({
          toastType: 'info',
          message: '处理已取消'
        })
        return
      }
    }

    // 完成处理
    showResult.value = true;
    totalTime.value = Math.round((Date.now() - startTime.value) / 1000);  // 计算总耗时(秒)
    updateLoadingChart();
    
    isWritingData.value = false;
    await bitable.ui.showToast({
      toastType: 'success',
      message: `数据获取完成: ${successCount.value}条成功，${failedCount.value}条失败`
    });
  } catch (error) {
    console.error("执行过程中出错:", error);
    isWritingData.value = false;
    showProgressDialog.value = false;
    
    await bitable.ui.showToast({
      toastType: 'error',
      message: error.message || '表格异常错误'
    });
  }
}

// 初始化进度图表
const initLoadingChart = async () => {
  await nextTick()
  const chartDom = document.getElementById('loadingChart')
  if (!chartDom) return

  loadingChartInstance.value = echarts.init(chartDom)

  const option = {
    backgroundColor: 'transparent',
    series: [{
      type: 'liquidFill',
      data: [
        {value: 0, itemStyle: {color: '#3370ff'}},
        {value: 0, itemStyle: {color: '#66a6ff'}},
        {value: 0, itemStyle: {color: '#89bdff'}}
      ],
      radius: '80%',
      amplitude: 20,
      waveAnimation: true,
      animationDuration: 2000,
      animationDurationUpdate: 1000,
      color: ['#3370ff', '#66a6ff', '#89bdff'],
      backgroundStyle: {
        color: '#f5f7fa'
      },
      outline: {
        show: true,
        borderDistance: 5,
        itemStyle: {
          color: 'none',
          borderColor: '#409eff',
          borderWidth: 2
        }
      },
      label: {
        show: true,
        fontSize: 30,
        fontWeight: 'bold',
        fontFamily: 'Arial',
        color: '#3370ff',
        insideColor: '#fff',
        formatter: () => {
          return `${Math.round(processedCount.value / totalCount.value * 100)}%`
        }
      }
    }]
  }

  loadingChartInstance.value.setOption(option)
}

// 更新进度图表
const updateLoadingChart = () => {
  if (!loadingChartInstance.value) return

  // 确保分母不为0
  const total = totalCount.value || 1
  const progress = Math.min(processedCount.value / total, 1)

  loadingChartInstance.value.setOption({
    series: [{
      data: [
        {value: progress, itemStyle: {color: '#3370ff'}},
        {value: progress * 0.85, itemStyle: {color: '#66a6ff'}},
        {value: progress * 0.7, itemStyle: {color: '#89bdff'}}
      ],
      label: {
        formatter: () => {
          return `${Math.round(progress * 100)}%`
        }
      }
    }]
  })
}

// 组件卸载时清理图表实例
onUnmounted(() => {
  if (loadingChartInstance.value) {
    loadingChartInstance.value.dispose()
  }
})

// -- 辅助算法区域
// --001== 获取所勾选字段的字段Id
const getSelectedFieldsId = (fieldList, checkedFields) => {
  const mappedFields = {};
  for (let field of checkedFields) {
    // 查找与checkedFields相匹配的fieldListSeView项目
    const foundField = fieldList.find(f => f.name ===  t(`selectGroup.videoInfo.${field}`));

    if ( ( field.endsWith('量') || field.endsWith('Count') ) && foundField && foundField.type !== 2)
      return ` [${t(`selectGroup.videoInfo.${field}`)}] ${t(`checks.number`)}` 
    else if (field == t('selectGroup.videoInfo.up主') && foundField && foundField.type !== 1)
      return ` [${t(`selectGroup.videoInfo.${field}`)}] ${t(`checks.text`)}`
    else if (field == t('selectGroup.videoInfo.发布时间') && foundField && foundField.type !== 5)
      return ` [${t(`selectGroup.videoInfo.${field}`)}] ${t(`checks.datetime`)}`
    else if ( ( field.endsWith('词云') || field.endsWith('Word Cloud') ) && foundField && foundField.type !== 17)
      return ` [${t(`selectGroup.videoInfo.${field}`)}] ${t(`checks.attachment`)}`

    // 如果找到了相应的项目，就使用其id，否则设置为-1
    mappedFields[field] = foundField ? foundField.id : -1;
  }
      
  return mappedFields;
} 


// --002== 请求在replit写的flask框架的接口，获取基本数据
/* @param:path
* get_bilibili_data-获取基本数据
* generate_comment_wordcloud-获取评论词云
* generate_barrage_wordcloud-获取弹幕词云
*/
const getbilibilidatabylink = async (path, videoLink) => {
  var data = qs.stringify({
    'video_url': videoLink
  });
  var url = `https://feishu-get-bilibili-data-backend-wuyi.replit.app/${path}`
  let res;

  if (path === 'get_bilibili_data') {
    var config = {
      method: 'post',
      url: url,
      data : data
    };

    await axios(config)
    .then(function (response) {
      console.log("getbilibilidatabylink() >> response.data || res", response.data);
      res = response.data
    })
    .catch(function (error) {
      console.log(error);
    });
    
    if (res.status === 200)
      return res.info
    else 
      return {"status": "-100", "result": res}
  } else if (path === 'generate_comment_wordcloud') {

    await axios.post(url, data, { responseType: 'blob' })
    .then(response => {
      // 将 Blob 转换为 File 对象
      const file = new File([response.data], "comment_wordcloud.png", { type: "image/png" });
      // 在这里可以根据需要处理 file 对象
      // 例如上传文件、添加到 FormData 或其他操作
      res = file
    })
    .catch(error => {
      console.error('Error:', error);
    });

    return res
  } else if (path === 'generate_barrage_wordcloud') {
    await axios.post(url, data, { responseType: 'blob' })
    .then(response => {
      // 将 Blob 转换为 File 对象
      const file = new File([response.data], "barrage_wordcloud.png", { type: "image/png" });
      // 在这里可以根据需要处理 file 对象
      // 例如上传文件、添加到 FormData 或其他操作
      res = file
    })
    .catch(error => {
      console.error('Error:', error);
    });

    return res
    } 
  
};  


// --003== 创建 mappedFieldIds 中 value 为 -1 的字段
const createFields = async () => {
  const selection = await bitable.base.getSelection()
  const table = await bitable.base.getTableById(selection.tableId)

  for (let key in mappedFieldIds.value) {
    if (mappedFieldIds.value[key] === -1) {
      switch (key) {
        case "title":  // 视频名称
          mappedFieldIds.value[key] = await table.addField({
            type: FieldType.Text,
            name: t(`selectGroup.videoInfo.title`),
          })
          break;
        case "uploader": 
          mappedFieldIds.value[key] = await table.addField({
            type: FieldType.Text,
            name: t(`selectGroup.videoInfo.uploader`),
          })
          break;
        case "releaseTime":
          mappedFieldIds.value[key] = await table.addField({
            type: FieldType.DateTime,
            name: t(`selectGroup.videoInfo.${key}`),
          })
          break;
        case "danmuCount":
        case "coinCount":
        case "viewCount":
        case "collectionCount":
        case "likeCount":
        case "commentCount":  // 评论量
        case "totalInterCount":  // 总互动量
        case "shareCount":
          mappedFieldIds.value[key] = await table.addField({
            type: FieldType.Number,
            name: t(`selectGroup.videoInfo.${key}`),
          })
          break;
        case "commentWc":
        case "danmuWc":
          mappedFieldIds.value[key] = await table.addField({
            type: FieldType.Attachment,
            name: t(`selectGroup.videoInfo.${key}`),
          })
          break;
        case "fetchDataTime":
          mappedFieldIds.value[key] = await table.addField({
            type: FieldType.DateTime,
            name: t(`selectGroup.videoInfo.${key}`),
          })
          break;
      }
    }
  }
}

// --004== 依据 recordId & filedId 获取 cell 值
const getCellValueByRFIDS = async (recordId, fieldId) => {
  const selection = await bitable.base.getSelection();
  const table = await bitable.base.getTableById(selection.tableId);
  const cellValue = await table.getCellValue(fieldId, recordId)

  // 处理cellValue为null或undefined的情况
  if (!cellValue) return null;
  
  if (typeof cellValue == 'object' && Array.isArray(cellValue) && cellValue.length > 0)
    return cellValue[0].text

  return cellValue
}

// --005== 依据 checkedFiledsToMap 做不同的请求处理
const getDataByCheckedFields = async(videoLink) => {
  let basicInfo = {};
  let commentWc;
  let barrageWc;

  if (
      checkedFieldsToMap.value.includes('uploader') ||
      checkedFieldsToMap.value.includes('releaseTime') ||
      checkedFieldsToMap.value.includes('danmuCount') ||
      checkedFieldsToMap.value.includes('coinCount') ||
      checkedFieldsToMap.value.includes('viewCount') ||
      checkedFieldsToMap.value.includes('collectionCount') ||
      checkedFieldsToMap.value.includes('likeCount') ||
      checkedFieldsToMap.value.includes('shareCount') || 
      checkedFieldsToMap.value.includes('commentCount') || 
      checkedFieldsToMap.value.includes('title') || 
      checkedFieldsToMap.value.includes('totalInterCount')     
    ) {
      basicInfo = await getbilibilidatabylink('get_bilibili_data', videoLink)
    }

    if (checkedFieldsToMap.value.includes('commentWc')) {
      commentWc = await getbilibilidatabylink('generate_comment_wordcloud', videoLink)
    }

    if (checkedFieldsToMap.value.includes('danmuWc')) {
      barrageWc = await getbilibilidatabylink('generate_barrage_wordcloud', videoLink)
    }


    return {basicInfo, commentWc, barrageWc}
}


// Map==全选事件
const handlecheckAllToMapChange = (val) => {
  const data = JSON.parse(JSON.stringify(fieldsToMap.value))
  console.log("val", val)
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
  console.log('checkedFieldsToMap:', checkedFieldsToMap.value)

}


onMounted(async () => {
  // 初始化勾选字段
  // const language = await bitable.bridge.getLanguage();

  // 获取字段列表 -- start
  const selection = await bitable.base.getSelection()
  const table = await bitable.base.getTableById(selection.tableId)
  const view = await table.getViewById(selection.viewId)
  fieldListSeView.value = await view.getFieldMetaList()
  console.log("onMounted >> 多维表格字段", fieldListSeView.value)
  console.log("onMounted >> 已选中的b站数据字段", checkedFieldsToMap.value)
  // 获取字段列表 -- end
  

  // 初始化可参与计算 "总交互量" 的对象数组，以"Count"结尾的
  allToCalcInterCount.value = fieldsToMap.value
    .filter(item => item.label.endsWith('Count'));

  console.log("onMounted >> allToCalcInterCount", allToCalcInterCount.value);

});
    
</script>

<style scoped>
.loading-step {
  margin: 20px 0;
  padding: 20px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
}

.loading-content {
  text-align: center;
}

.loading-chart {
  width: 240px;
  height: 240px;
  margin: 0 auto 16px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.08);
  border-radius: 50%;
}

.loading-text {
  font-size: 14px;
  color: #606266;
  line-height: 1.5;
}

.time-info {
  margin-top: 8px;
  font-size: 12px;
  color: #909399;
}

.dialog-footer {
  display: flex;
  justify-content: flex-end;
  margin-top: 16px;
}
</style>
