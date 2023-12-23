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
    <el-alert style="display: flex;align-items: flex-start;margin: 20px 0;background-color: #e1eaff;color: #606266;" :title="$t('alerts.selectGroupFieldTip')" type="info" show-icon />


    <!-- 选择对应的字段映射 -->


    <!-- 提交按钮 -->
    <el-button v-loading="isWritingData" @click="writeData" :disabled="!issubmitAbled" color="#3370ff" type="primary" plain size="large">{{ $t('submit') }}</el-button>
  </el-form>
</template>

<script setup>
import { bitable, FieldType } from '@lark-base-open/js-sdk';
import { useI18n } from 'vue-i18n';
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';
import qs from 'qs';


// -- 数据区域
const { t } = useI18n();
const fieldListSeView = ref([])
const linkFieldId = ref('')  // 链接字段Id

const isWritingData = ref(false)

const checkAllToMap = ref(false)
const isIndeterminateToMap = ref(true)
const fieldsToMap = ref([
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
    "label": "commentWc"
  },
  {
    "label": "danmuWc"
  },
  {
    "label": "fetchDataTime"
  }
])   //  可以创建的字段
const checkedFieldsToMap = ref([
  'uploader',
  'releaseTime',
  'danmuCount',
  'coinCount',
  'viewCount',
  'collectionCount',
  'likeCount',
  'shareCount',
  'fetchDataTime'
])   // 默认的to-map的字段

const issubmitAbled = computed(() => {
  return linkFieldId.value && checkedFieldsToMap.value.length
})  // 是否允许提交，及必选字段是否都填写

const mappedFieldIds = ref({})


// -- 核心算法区域
// --001== 写入数据
const writeData = async () => {
  // 获取字段数据Ids object类型
  isWritingData.value = true

  {
    const mappedFields = getSelectedFieldsId(fieldListSeView.value, checkedFieldsToMap.value)
    console.log("writeData() >> original mappedFields", mappedFields)
    if (typeof mappedFields == 'string') // 错误处理，提示格式错误 
    {
      await bitable.ui.showToast({
        toastType: 'warning',
        message: mappedFields
      })
      return
    }

    // 创建缺少的字段
    mappedFieldIds.value = mappedFields
    await createFields()

    console.log("writeData() >> finished mappedFieldIds", mappedFieldIds.value)
  }
  // 加载bitable实例
  const { tableId, viewId } = await bitable.base.getSelection();
  const table = await bitable.base.getActiveTable();
  const view = await table.getViewById(viewId);

  // ## mode1: 全部记录
  const RecordList = await view.getVisibleRecordIdList()

  // ## model2: 交互式选择记录 
  // const RecordList = await bitable.ui.selectRecordIdList(tableId, viewId);

  for (let recordId of RecordList) {
    // TODO：在这里书写处理逻辑——数据请求、数据写入等
    // 非空处理
    console.log("writeData() >> recordId", recordId)
    
    let videoLink
    try {
      videoLink = await getCellValueByRFIDS(recordId, linkFieldId.value)
    } catch (error) {
      continue
      
    }
    console.log("writeData() >> videoLink", videoLink)


    const totalVideoInfo = await getDataByCheckedFields(videoLink)
    console.log("writeData() >> totalVideoInfo", totalVideoInfo)

    for (let field of checkedFieldsToMap.value) {
      console.log("field", field)
      if (field == 'uploader') {  // up主字段，Text 格式
        await table.setCellValue(mappedFieldIds.value[field], recordId, [{ type: 'text', text: totalVideoInfo.basicInfo[field] }])
      } else if (field.endsWith('Count')) {  // xx量字段，Number格式
        await table.setCellValue(mappedFieldIds.value[field], recordId, totalVideoInfo.basicInfo[field])
      } else if (field.endsWith('commentWc')) {  // 评论词云，附件格式
        const attachmentField = await table.getFieldById(mappedFieldIds.value[field])
        attachmentField.setValue(recordId, totalVideoInfo.commentWc)
      } else if (field.endsWith('danmuWc')) {  // 弹幕词云，附件格式
        const attachmentField = await table.getFieldById(mappedFieldIds.value[field])
        attachmentField.setValue(recordId, totalVideoInfo.barrageWc)
      } else if (field === 'releaseTime') { // 发布时间，datetime格式
        const datetimeFieldId = await table.getFieldById(mappedFieldIds.value[field])
        datetimeFieldId.setValue(recordId, totalVideoInfo.basicInfo[field])
      } else if (field === 'fetchDataTime') { // 发布时间，datetime格式
        const datetimeFieldId = await table.getFieldById(mappedFieldIds.value[field])
        datetimeFieldId.setValue(recordId, Date.now())
      }
    }

  }

  isWritingData.value = false
  await bitable.ui.showToast({
    toastType: 'success',
    message: '数据获取完成'
  })

  
  
  
}





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
  var url = `https://getbilibilidatabylink.1326906378.repl.co/${path}`
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

  if (typeof cellValue == 'object')
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
      checkedFieldsToMap.value.includes('shareCount')
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
  console.log("onMounted >> fieldListSeView", fieldListSeView.value)
  console.log("onMounted >> checkedFieldsToMap", checkedFieldsToMap.value)
  
  // const res = await getDataByCheckedFields('https://www.bilibili.com/video/BV15w411T7Jf/')
  // console.log("onMounted() >> getDataByCheckedFields", res)


  // 获取字段列表 -- end
  
  // 从缓存中获取数据 -- start
  // if (localStorage.getItem('ossConfig') !== null) {
  //   ossConfig.value = JSON.parse(localStorage.getItem('ossConfig')) 
  // }
  // if (localStorage.getItem('attchImgFieldId') !== null) {
  //   attchImgFieldId.value = localStorage.getItem('attchImgFieldId')
  // }
  // if (localStorage.getItem('linkFieldId') !== null) {
  //   linkFieldId.value = localStorage.getItem('linkFieldId')
  // }
  // 从缓存中获取数据 -- end
});
    
</script>



<style scoped>
  
</style>
