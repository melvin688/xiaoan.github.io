<template>
  <div class="page-container">
    <!-- 顶部操作栏 -->
    <div class="header-actions">
      <div class="header-title">
        <span>{{ $t('tables.title') }}</span>
        <el-tag type="info" style="margin-left: 12px;">{{ tableList.length }} {{ $t('tables.tables') }}</el-tag>
      </div>
      <el-button type="primary" :icon="Plus" @click="handleAdd">
        {{ $t('tables.addTable') }}
      </el-button>
    </div>

    <!-- 桌台卡片网格 -->
    <div class="tables-grid" v-loading="loading">
      <el-card
        v-for="table in tableList"
        :key="table.id"
        class="table-card"
        :class="{ 'occupied': table.current_orders && table.current_orders.length > 0 }"
      >
        <!-- 卡片头部 -->
        <template #header>
          <div class="table-card-header">
            <div class="table-info">
              <span class="table-number">{{ table.table_number }}</span>
              <span class="table-name">{{ table.table_name }}</span>
            </div>
            <div class="table-actions">
              <el-dropdown @command="(cmd) => handleTableAction(cmd, table)">
                <el-button :icon="More" circle size="small" />
                <template #dropdown>
                  <el-dropdown-menu>
                    <el-dropdown-item command="edit">
                      <el-icon><Edit /></el-icon>
                      {{ $t('common.edit') }}
                    </el-dropdown-item>
                    <el-dropdown-item command="qr">
                      <el-icon><CoffeeCup /></el-icon>
                      {{ $t('tables.viewQRCode') }}
                    </el-dropdown-item>
                    <el-dropdown-item command="delete" divided>
                      <el-icon><Delete /></el-icon>
                      {{ $t('common.delete') }}
                    </el-dropdown-item>
                  </el-dropdown-menu>
                </template>
              </el-dropdown>
            </div>
          </div>
        </template>

        <!-- 卡片内容 -->
        <div class="table-card-body">
          <!-- 状态和容量 -->
          <div class="table-status">
            <el-tag :type="(table.current_orders && table.current_orders.length > 0) ? 'warning' : 'success'" size="large">
              {{ (table.current_orders && table.current_orders.length > 0) ? $t('tables.statusOptions.occupied') : $t('tables.statusOptions.available') }}
            </el-tag>
          </div>

          <!-- 座位数量调整 -->
          <div class="capacity-control">
            <div class="capacity-label">
              <el-icon><User /></el-icon>
              <span>{{ $t('tables.capacity') }}</span>
            </div>
            <div class="capacity-adjuster">
              <el-button
                :icon="Minus"
                circle
                size="small"
                @click="adjustCapacity(table, -1)"
                :disabled="table.capacity <= 1"
              />
              <span class="capacity-number">{{ table.capacity }}</span>
              <el-button
                :icon="Plus"
                circle
                size="small"
                @click="adjustCapacity(table, 1)"
                :disabled="table.capacity >= 20"
              />
            </div>
          </div>

          <!-- 当前订单 -->
          <div class="current-orders" v-if="table.current_orders && table.current_orders.length > 0">
            <div class="orders-header">
              <el-icon><ShoppingCart /></el-icon>
              <span>{{ $t('tables.currentOrders') }}</span>
            </div>
            <div class="orders-list">
              <div
                v-for="order in table.current_orders"
                :key="order.id"
                class="order-item"
                @click="viewOrderDetail(order)"
              >
                <div class="order-header">
                  <span class="order-number">#{{ order.order_no }}</span>
                  <el-tag :type="getServiceTypeTag(order.service_type)" size="small" style="margin-left: 8px;">
                    {{ getServiceTypeLabel(order.service_type) }}
                  </el-tag>
                  <el-tag :type="getPaymentMethodTag(order.payment_method)" size="small" style="margin-left: 4px;">
                    {{ getPaymentMethodLabel(order.payment_method) }}
                  </el-tag>
                  <el-tag :type="getOrderStatusType(order.status)" size="small">
                    {{ $t(`orders.status.${order.status}`) }}
                  </el-tag>
                </div>
                <div class="order-service-info" v-if="order.service_type !== 'dine-in' && order.remark">
                  <span class="service-info-label">{{ getServiceInfoLabel(order.service_type) }}:</span>
                  <span class="service-info-value">{{ order.remark }}</span>
                </div>
                <div class="order-items-preview" v-if="order.items && order.items.length > 0">
                  <div v-for="item in order.items" :key="item.id" class="item-preview">
                    <span class="item-name">{{ getProductName(item) }}</span>
                    <span class="item-qty">x{{ item.quantity }}</span>
                  </div>
                </div>
                <div class="order-amount">
                  <span class="amount-label">总计：</span>
                  <span class="amount-value">{{ order.total_amount }}</span>
                </div>
              </div>
            </div>
            <!-- 批量操作按钮 -->
            <div class="batch-actions">
              <el-button 
                type="primary" 
                size="small"
                :icon="Printer"
                @click.stop="handlePrintAllOrders(table)"
                :loading="table.printing"
                class="print-btn"
              >
                打印全部订单
              </el-button>
              <el-button 
                type="danger" 
                size="small"
                :icon="Delete"
                @click.stop="handleClearAllOrders(table)"
                :loading="table.clearing"
                class="clear-btn"
              >
                清空全部订单
              </el-button>
            </div>
          </div>
          <div v-else class="no-orders">
            <el-icon><CoffeeCup /></el-icon>
            <span>{{ $t('tables.noOrders') }}</span>
          </div>
        </div>
      </el-card>
    </div>

    <!-- 空状态 -->
    <el-empty v-if="!loading && tableList.length === 0" :description="$t('tables.noTables')" />

    <!-- 添加/编辑对话框 -->
    <el-dialog
      v-model="dialogVisible"
      :title="dialogTitle"
      width="500px"
    >
      <el-form :model="formData" :rules="rules" ref="formRef" label-width="120px">
        <el-form-item :label="$t('tables.tableNumber')" prop="table_number">
          <el-input v-model="formData.table_number" :placeholder="$t('tables.tableNumberPlaceholder')" />
        </el-form-item>
        <el-form-item :label="$t('tables.tableName')" prop="table_name">
          <el-input v-model="formData.table_name" :placeholder="$t('tables.tableNamePlaceholder')" />
        </el-form-item>
        <el-form-item :label="$t('tables.capacity')" prop="capacity">
          <el-input-number v-model="formData.capacity" :min="1" :max="20" />
        </el-form-item>
        <el-form-item :label="$t('tables.status')" prop="status">
          <el-select v-model="formData.status" style="width: 100%">
            <el-option value="available" :label="$t('tables.statusOptions.available')" />
            <el-option value="occupied" :label="$t('tables.statusOptions.occupied')" />
          </el-select>
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="dialogVisible = false">{{ $t('common.cancel') }}</el-button>
        <el-button type="primary" @click="handleSubmit">{{ $t('common.confirm') }}</el-button>
      </template>
    </el-dialog>

    <!-- 二维码对话框 -->
    <el-dialog
      v-model="qrDialogVisible"
      :title="$t('tables.qrCode')"
      width="400px"
    >
      <div style="text-align: center;">
        <div style="font-size: 18px; margin-bottom: 20px; font-weight: 600; color: #6B4423;">
          {{ currentTable?.table_name }} ({{ currentTable?.table_number }})
        </div>
        <div id="qrcode" style="display: flex; justify-content: center;"></div>
        <div style="margin-top: 20px; color: #8D6E63; font-size: 12px;">
          {{ currentTable?.qr_code }}
        </div>
      </div>
    </el-dialog>

    <!-- 订单详情对话框 -->
    <el-dialog
      v-model="orderDetailVisible"
      :title="$t('orders.orderDetail')"
      width="600px"
    >
      <div v-if="currentOrder" class="order-detail">
        <div class="detail-row">
          <span class="detail-label">{{ $t('orders.orderNumber') }}:</span>
          <span class="detail-value">#{{ currentOrder.order_no }}</span>
        </div>
        <div class="detail-row">
          <span class="detail-label">{{ $t('orders.orderTime') }}:</span>
          <span class="detail-value">{{ formatTime(currentOrder.created_at) }}</span>
        </div>
        <div class="detail-row">
          <span class="detail-label">{{ $t('orders.status') }}:</span>
          <el-tag :type="getOrderStatusType(currentOrder.status)">
            {{ $t(`orders.status.${currentOrder.status}`) }}
          </el-tag>
        </div>
        <el-divider />
        <div class="order-items">
          <h4>{{ $t('orders.items') }}</h4>
          <div v-for="(item, index) in currentOrder.items" :key="item.id" class="order-item-detail">
            <span class="item-name">{{ getProductName(item) }}</span>
            <span class="item-quantity">x{{ item.quantity }}</span>
            <span class="item-price">K {{ item.unit_price * item.quantity }}</span>
            <el-button 
              type="danger" 
              :icon="Delete" 
              circle 
              size="small"
              @click="deleteOrderItem(item, index)"
              title="删除商品"
            />
          </div>
        </div>
        <el-divider />
        <div class="order-total">
          <span class="total-label">{{ $t('orders.total') }}:</span>
          <span class="total-amount">{{ currentOrder.total_amount }}</span>
        </div>
        <div class="order-actions">
          <el-button
            v-if="currentOrder.status === 'pending'"
            type="primary"
            @click="updateStatus(currentOrder, 'preparing')"
          >
            {{ $t('orders.actions.start') }}
          </el-button>
          <el-button
            v-if="currentOrder.status === 'preparing'"
            type="success"
            @click="updateStatus(currentOrder, 'completed')"
          >
            {{ $t('orders.actions.complete') }}
          </el-button>
          <el-button @click="printOrder(currentOrder)">
            {{ $t('orders.actions.print') }}
          </el-button>
        </div>
      </div>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, nextTick } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import {
  Plus,
  Minus,
  More,
  Edit,
  Delete,
  User,
  ShoppingCart,
  CoffeeCup,
  Printer
} from '@element-plus/icons-vue'
import { useI18n } from 'vue-i18n'
import { getTables, createTable, updateTable, deleteTable } from '../api/tables'
import { getTableOrders, updateOrderStatus, printOrder as printOrderApi, deleteOrderItem as deleteOrderItemApi, deleteOrder } from '../api/orders'
import QRCode from 'qrcode'
import moment from 'moment'

const { t, locale } = useI18n()

const loading = ref(false)
const tableList = ref([])
const dialogVisible = ref(false)
const qrDialogVisible = ref(false)
const orderDetailVisible = ref(false)
const formRef = ref(null)
const currentTable = ref(null)
const currentOrder = ref(null)
const isEdit = ref(false)

const formData = ref({
  table_number: '',
  table_name: '',
  capacity: 4,
  status: 'available'
})

const rules = {
  table_number: [
    { required: true, message: t('tables.tableNumberRequired'), trigger: 'blur' }
  ],
  table_name: [
    { required: true, message: t('tables.tableNameRequired'), trigger: 'blur' }
  ]
}

const dialogTitle = computed(() => {
  return isEdit.value ? t('tables.editTable') : t('tables.addTable')
})

// 获取餐桌列表及订单信息
const fetchTables = async () => {
  loading.value = true
  try {
    const res = await getTables()
    if (res.success) {
      tableList.value = res.data
      // 为每张桌子获取当前订单
      await fetchTableOrders()
    }
  } catch (error) {
    ElMessage.error(t('common.loadFailed'))
  } finally {
    loading.value = false
  }
}

// 获取所有桌台的订单
const fetchTableOrders = async () => {
  for (let table of tableList.value) {
    try {
      const res = await getTableOrders(table.id)
      if (res.success) {
        table.current_orders = res.data || []
      }
    } catch (error) {
      console.error(`Failed to fetch orders for table ${table.id}:`, error)
      table.current_orders = []
    }
  }
}

// 调整座位数
const adjustCapacity = async (table, delta) => {
  const newCapacity = table.capacity + delta
  if (newCapacity < 1 || newCapacity > 20) return

  try {
    const res = await updateTable(table.id, {
      ...table,
      capacity: newCapacity
    })
    
    if (res.success) {
      table.capacity = newCapacity
      ElMessage.success(t('common.updateSuccess'))
    }
  } catch (error) {
    ElMessage.error(error.message || t('common.operationFailed'))
  }
}

// 处理桌台操作
const handleTableAction = (command, table) => {
  switch (command) {
    case 'edit':
      handleEdit(table)
      break
    case 'qr':
      handleViewQR(table)
      break
    case 'delete':
      handleDelete(table)
      break
  }
}

// 添加餐桌
const handleAdd = () => {
  isEdit.value = false
  formData.value = {
    table_number: '',
    table_name: '',
    capacity: 4,
    status: 'available'
  }
  dialogVisible.value = true
}

// 编辑餐桌
const handleEdit = (row) => {
  isEdit.value = true
  formData.value = { ...row }
  dialogVisible.value = true
}

// 提交表单
const handleSubmit = async () => {
  if (!formRef.value) return
  
  await formRef.value.validate(async (valid) => {
    if (valid) {
      try {
        if (isEdit.value) {
          const res = await updateTable(formData.value.id, formData.value)
          if (res.success) {
            ElMessage.success(t('common.updateSuccess'))
            dialogVisible.value = false
            fetchTables()
          }
        } else {
          const res = await createTable(formData.value)
          if (res.success) {
            ElMessage.success(t('common.createSuccess'))
            dialogVisible.value = false
            fetchTables()
          }
        }
      } catch (error) {
        ElMessage.error(error.message || t('common.operationFailed'))
      }
    }
  })
}

// 删除餐桌
const handleDelete = async (row) => {
  try {
    await ElMessageBox.confirm(
      t('tables.deleteConfirm'),
      t('common.warning'),
      {
        confirmButtonText: t('common.confirm'),
        cancelButtonText: t('common.cancel'),
        type: 'warning'
      }
    )
    
    const res = await deleteTable(row.id)
    if (res.success) {
      ElMessage.success(t('common.deleteSuccess'))
      fetchTables()
    }
  } catch (error) {
    if (error !== 'cancel') {
      ElMessage.error(error.message || t('common.operationFailed'))
    }
  }
}

// 查看二维码
const handleViewQR = (row) => {
  currentTable.value = row
  qrDialogVisible.value = true
  
  nextTick(async () => {
    // 清空之前的二维码
    const qrcodeEl = document.getElementById('qrcode')
    qrcodeEl.innerHTML = ''
    
    try {
      // 生成新的二维码
      const canvas = document.createElement('canvas')
      await QRCode.toCanvas(canvas, row.qr_code, {
        width: 256,
        margin: 2,
        color: {
          dark: '#000000',
          light: '#ffffff'
        }
      })
      qrcodeEl.appendChild(canvas)
    } catch (error) {
      console.error('QR code generation failed:', error)
      ElMessage.error('二维码生成失败')
    }
  })
}

// 查看订单详情
const viewOrderDetail = (order) => {
  currentOrder.value = order
  orderDetailVisible.value = true
}

// 更新订单状态
const updateStatus = async (order, status) => {
  try {
    const res = await updateOrderStatus(order.id, status)
    if (res.success) {
      ElMessage.success(t('orders.messages.statusUpdated'))
      order.status = status
      orderDetailVisible.value = false
      fetchTables()
    }
  } catch (error) {
    ElMessage.error(error.message || t('orders.messages.statusFailed'))
  }
}

// 打印订单
const printOrder = async (order) => {
  try {
    const res = await printOrderApi(order.id)
    if (res.success) {
      // 显示打印预览对话框
      const receipt = res.data.receipt
      
      // 创建打印窗口
      const printWindow = window.open('', '', 'width=400,height=600')
      if (!printWindow) {
        ElMessage.error('无法打开打印窗口,请允许弹窗')
        return
      }
      
      printWindow.document.write(`
        <html>
          <head>
            <meta charset="UTF-8">
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <title>打印小票</title>
            <style>
              @page { size: 80mm auto; margin: 0; }
              @media print {
                body { margin: 0; padding: 0; }
                .no-print { display: none; }
              }
              body { 
                font-family: 'Courier New', monospace;
                font-size: 12px;
                width: 80mm;
                margin: 0;
                padding: 5mm;
                line-height: 1.6;
                white-space: pre-wrap;
              }
              .print-actions {
                text-align: center;
                margin: 20px 0;
                padding: 10px;
                background: #f5f5f5;
              }
              button {
                margin: 0 5px;
                padding: 8px 16px;
                font-size: 14px;
                cursor: pointer;
                border-radius: 4px;
                border: 1px solid #ddd;
              }
              .print-btn {
                background: #409EFF;
                color: white;
                border-color: #409EFF;
              }
              .close-btn {
                background: white;
              }
            </style>
          </head>
          <body>
            <div class="print-actions no-print">
              <button class="print-btn" onclick="window.print()">打印</button>
              <button class="close-btn" onclick="window.close()">关闭</button>
            </div>
            <pre>${receipt}</pre>
          </body>
        </html>
      `)
      printWindow.document.close()
      
      ElMessage.success(t('orders.messages.printed'))
    }
  } catch (error) {
    console.error('打印错误:', error)
    ElMessage.error(error.message || t('orders.messages.printFailed'))
  }
}

// 打印桌台所有订单
const handlePrintAllOrders = async (table) => {
  if (!table.current_orders || table.current_orders.length === 0) {
    ElMessage.warning('该餐桌没有订单')
    return
  }

  try {
    // 设置打印状态
    table.printing = true

    // 收集所有订单ID
    const orderIds = table.current_orders.map(order => order.id)
    
    // 批量打印订单
    const printRes = await fetch(`${import.meta.env.VITE_API_BASE_URL || 'http://localhost:3000'}/api/print/batch`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${localStorage.getItem('admin_token')}`
      },
      body: JSON.stringify({ orderIds })
    })

    const printData = await printRes.json()

    if (printData.success) {
      // 检测设备类型
      const isAndroid = /Android/i.test(navigator.userAgent)

      if (isAndroid) {
        // 安卓设备：提供多种打印选项
        await handleAndroidPrint(printData.data)
      } else {
        // 桌面设备：使用传统打印窗口
        for (const receipt of printData.data) {
          // 创建打印窗口
          const printWindow = window.open('', '', 'width=400,height=600')
          printWindow.document.write(`
            <html>
              <head>
                <meta charset="UTF-8">
                <meta name="viewport" content="width=device-width, initial-scale=1.0">
                <style>
                  @page { size: 80mm auto; margin: 0; }
                  @media print {
                    body { margin: 0; padding: 0; }
                  }
                  body { 
                    font-family: 'Courier New', monospace;
                    font-size: 12px;
                    width: 80mm;
                    margin: 0;
                    padding: 5mm;
                    line-height: 1.6;
                    white-space: pre-wrap;
                  }
                </style>
              </head>
              <body>${receipt.content}</body>
            </html>
          `)
          printWindow.document.close()
          
          // 延迟打印以确保内容加载完成
          setTimeout(() => {
            printWindow.print()
            setTimeout(() => printWindow.close(), 100)
          }, 500)
        }

        // 等待打印窗口处理
        await new Promise(resolve => setTimeout(resolve, 1000))
      }

      ElMessage.success(`已成功打印 ${orderIds.length} 个订单`)
    } else {
      throw new Error(printData.message || '打印失败')
    }
  } catch (error) {
    ElMessage.error(error.message || '打印失败')
  } finally {
    table.printing = false
  }
}

// 清空桌台所有订单
const handleClearAllOrders = async (table) => {
  if (!table.current_orders || table.current_orders.length === 0) {
    ElMessage.warning('该餐桌没有订单')
    return
  }

  try {
    await ElMessageBox.confirm(
      `确定要删除 ${table.table_name} 的所有 ${table.current_orders.length} 个订单吗？此操作不可恢复！`,
      '删除订单确认',
      {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'error',
        distinguishCancelAndClose: true
      }
    )

    // 设置清空状态
    table.clearing = true

    // 收集所有订单ID
    const orderIds = table.current_orders.map(order => order.id)

    // 批量删除订单
    for (const orderId of orderIds) {
      await deleteOrder(orderId)
    }

    ElMessage.success(`已删除 ${orderIds.length} 个订单`)
    
    // 刷新餐桌数据
    await fetchTables()
  } catch (error) {
    if (error !== 'cancel' && error !== 'close') {
      ElMessage.error(error.message || '操作失败')
    }
  } finally {
    table.clearing = false
  }
}

// 一键打印并清空餐桌所有订单（保留以兼容）
const handlePrintAndClearTable = async (table) => {
  if (!table.current_orders || table.current_orders.length === 0) {
    ElMessage.warning('该餐桌没有订单')
    return
  }

  try {
    await ElMessageBox.confirm(
      `确定要打印 ${table.table_name} 的所有 ${table.current_orders.length} 个订单并清空吗？`,
      '打印并清空确认',
      {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning',
        distinguishCancelAndClose: true
      }
    )

    // 设置打印状态
    table.printing = true

    // 收集所有订单ID
    const orderIds = table.current_orders.map(order => order.id)
    
    // 批量打印订单
    const printRes = await fetch(`${import.meta.env.VITE_API_BASE_URL || 'http://localhost:3000'}/api/print/batch`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${localStorage.getItem('admin_token')}`
      },
      body: JSON.stringify({ orderIds })
    })

    const printData = await printRes.json()

    if (printData.success) {
      // 检测设备类型
      const isAndroid = /Android/i.test(navigator.userAgent)
      const isIOS = /iPhone|iPad|iPod/i.test(navigator.userAgent)
      const isMobile = isAndroid || isIOS

      if (isAndroid) {
        // 安卓设备：提供多种打印选项
        await handleAndroidPrint(printData.data)
      } else {
        // 桌面设备：使用传统打印窗口
        for (const receipt of printData.data) {
          // 创建打印窗口
          const printWindow = window.open('', '', 'width=400,height=600')
          printWindow.document.write(`
            <html>
              <head>
                <meta charset="UTF-8">
                <meta name="viewport" content="width=device-width, initial-scale=1.0">
                <style>
                  @page { size: 80mm auto; margin: 0; }
                  @media print {
                    body { margin: 0; padding: 0; }
                  }
                  body { 
                    font-family: 'Courier New', monospace;
                    font-size: 12px;
                    width: 80mm;
                    margin: 0;
                    padding: 5mm;
                    line-height: 1.6;
                    white-space: pre-wrap;
                  }
                </style>
              </head>
              <body>${receipt.content}</body>
            </html>
          `)
          printWindow.document.close()
          
          // 延迟打印以确保内容加载完成
          setTimeout(() => {
            printWindow.print()
            setTimeout(() => printWindow.close(), 100)
          }, 500)
        }

        // 等待打印窗口处理
        await new Promise(resolve => setTimeout(resolve, 1000))
      }

      // 批量更新订单状态为已完成
      for (const orderId of orderIds) {
        await updateOrderStatus(orderId, 'completed')
      }

      ElMessage.success(`已成功打印 ${orderIds.length} 个订单并清空`)
      
      // 刷新餐桌数据
      await fetchTables()
    } else {
      throw new Error(printData.message || '打印失败')
    }
  } catch (error) {
    if (error !== 'cancel' && error !== 'close') {
      ElMessage.error(error.message || '操作失败')
    }
  } finally {
    table.printing = false
  }
}

// 安卓设备打印处理
const handleAndroidPrint = async (receipts) => {
  return new Promise((resolve, reject) => {
    // 创建打印选项对话框
    ElMessageBox({
      title: '选择打印方式',
      message: `
        <div style="padding: 20px;">
          <p style="margin-bottom: 15px;">检测到安卓设备，请选择打印方式：</p>
          <button id="print-browser" style="width: 100%; padding: 12px; margin-bottom: 10px; border: 1px solid #ddd; border-radius: 8px; background: #fff; cursor: pointer;">
            📄 浏览器打印
          </button>
          <button id="print-app" style="width: 100%; padding: 12px; margin-bottom: 10px; border: 1px solid #ddd; border-radius: 8px; background: #fff; cursor: pointer;">
            📱 打印APP (RawBT)
          </button>
          <button id="print-share" style="width: 100%; padding: 12px; margin-bottom: 10px; border: 1px solid #ddd; border-radius: 8px; background: #fff; cursor: pointer;">
            📤 分享到打印APP
          </button>
          <button id="print-download" style="width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 8px; background: #fff; cursor: pointer;">
            💾 下载为文本文件
          </button>
        </div>
      `,
      dangerouslyUseHTMLString: true,
      showConfirmButton: false,
      showCancelButton: true,
      cancelButtonText: '取消'
    }).then(() => {
      resolve()
    }).catch(() => {
      reject('cancel')
    })

    // 等待DOM渲染
    setTimeout(() => {
      // 浏览器打印
      document.getElementById('print-browser')?.addEventListener('click', () => {
        receipts.forEach(receipt => {
          const printWindow = window.open('', '', 'width=device-width')
          printWindow.document.write(`
            <html>
              <head>
                <meta charset="UTF-8">
                <meta name="viewport" content="width=device-width, initial-scale=1.0">
                <style>
                  @page { size: auto; margin: 5mm; }
                  body { 
                    font-family: monospace;
                    font-size: 14px;
                    line-height: 1.6;
                    white-space: pre-wrap;
                    padding: 10px;
                  }
                </style>
              </head>
              <body>${receipt.content}</body>
            </html>
          `)
          printWindow.document.close()
          setTimeout(() => printWindow.print(), 300)
        })
        ElMessageBox.close()
        resolve()
      })

      // 打印APP (RawBT)
      document.getElementById('print-app')?.addEventListener('click', () => {
        const allContent = receipts.map(r => r.content).join('\n\n---\n\n')
        const rawbtIntent = `intent://print#Intent;scheme=rawbt;package=ru.a402d.rawbtprinter;S.text=${encodeURIComponent(allContent)};end`
        
        try {
          window.location.href = rawbtIntent
          ElMessage.success('正在打开打印APP...')
        } catch (e) {
          ElMessage.error('请先安装 RawBT 打印APP')
        }
        ElMessageBox.close()
        resolve()
      })

      // 分享到打印APP
      document.getElementById('print-share')?.addEventListener('click', () => {
        const allContent = receipts.map(r => r.content).join('\n\n---\n\n')
        
        if (navigator.share) {
          navigator.share({
            title: 'Alisa Cake 订单',
            text: allContent
          }).then(() => {
            ElMessage.success('已分享到打印APP')
          }).catch(() => {
            // 降级：复制到剪贴板
            copyToClipboard(allContent)
          })
        } else {
          copyToClipboard(allContent)
        }
        ElMessageBox.close()
        resolve()
      })

      // 下载为文本文件
      document.getElementById('print-download')?.addEventListener('click', () => {
        const allContent = receipts.map(r => r.content).join('\n\n---\n\n')
        const blob = new Blob([allContent], { type: 'text/plain;charset=utf-8' })
        const url = URL.createObjectURL(blob)
        const a = document.createElement('a')
        a.href = url
        a.download = `orders_${Date.now()}.txt`
        a.click()
        URL.revokeObjectURL(url)
        ElMessage.success('文件已下载，可在打印APP中打开')
        ElMessageBox.close()
        resolve()
      })
    }, 100)
  })
}

// 复制到剪贴板
const copyToClipboard = (text) => {
  if (navigator.clipboard) {
    navigator.clipboard.writeText(text).then(() => {
      ElMessage.success('内容已复制到剪贴板，可粘贴到打印APP')
    })
  } else {
    // 降级方案
    const textarea = document.createElement('textarea')
    textarea.value = text
    textarea.style.position = 'fixed'
    textarea.style.opacity = '0'
    document.body.appendChild(textarea)
    textarea.select()
    document.execCommand('copy')
    document.body.removeChild(textarea)
    ElMessage.success('内容已复制到剪贴板')
  }
}

// 获取订单状态类型
const getOrderStatusType = (status) => {
  const typeMap = {
    pending: 'warning',
    preparing: 'primary',
    completed: 'success',
    cancelled: 'info'
  }
  return typeMap[status] || 'info'
}

// 获取服务类型标签样式
const getServiceTypeTag = (serviceType) => {
  const typeMap = {
    'dine-in': 'success',
    'delivery': 'warning',
    'takeaway': 'primary'
  }
  return typeMap[serviceType] || 'info'
}

// 获取服务类型标签文本
const getServiceTypeLabel = (serviceType) => {
  const labelMap = {
    'dine-in': '堂食',
    'delivery': '外卖',
    'takeaway': '自取'
  }
  return labelMap[serviceType] || '堂食'
}

// 获取服务信息标签
const getServiceInfoLabel = (serviceType) => {
  const labelMap = {
    'delivery': '配送地址',
    'takeaway': '取餐时间'
  }
  return labelMap[serviceType] || '备注'
}

// 获取支付方式标签样式
const getPaymentMethodTag = (paymentMethod) => {
  const typeMap = {
    'kpay': 'success',
    'cash': 'warning'
  }
  return typeMap[paymentMethod] || 'info'
}

// 获取支付方式标签文本
const getPaymentMethodLabel = (paymentMethod) => {
  const labelMap = {
    'kpay': 'KBZPAY',
    'cash': '现金'
  }
  return labelMap[paymentMethod] || '现金'
}

// 格式化时间
const formatTime = (time) => {
  return moment(time).format('YYYY-MM-DD HH:mm:ss')
}

// 获取商品名称（根据当前语言）
const getProductName = (item) => {
  if (!item) return ''
  
  const lang = locale.value
  if (lang === 'zh') return item.name_zh || item.product_name
  if (lang === 'en') return item.name_en || item.product_name
  return item.product_name
}

// 删除订单商品
const deleteOrderItem = async (item, index) => {
  try {
    await ElMessageBox.confirm(
      `确定要删除商品 "${getProductName(item)}" 吗？`,
      '删除商品',
      {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }
    )
    
    // 调用后端API删除商品
    const res = await deleteOrderItemApi(item.id)
    
    if (res.success) {
      // 从当前订单中移除该商品
      currentOrder.value.items.splice(index, 1)
      
      // 更新订单总金额
      currentOrder.value.total_amount = res.data.new_total
      
      // 如果订单被取消了（没有商品了）
      if (res.data.order_cancelled) {
        ElMessage.warning('订单所有商品已删除，订单已取消')
        orderDetailVisible.value = false
      } else {
        ElMessage.success('商品已删除')
      }
      
      // 刷新订单数据
      fetchTables()
    }
  } catch (error) {
    // 用户取消删除
    if (error !== 'cancel') {
      ElMessage.error(error.message || '删除失败')
    }
  }
}

onMounted(() => {
  fetchTables()
  // 每30秒刷新一次订单数据
  setInterval(() => {
    fetchTableOrders()
  }, 30000)
})
</script>

<style scoped lang="scss">
.page-container {
  padding: 20px;
  background: linear-gradient(180deg, #FFF8F0 0%, #F5F5F5 100%);
  min-height: calc(100vh - 60px);
}

// 顶部操作栏
.header-actions {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
  padding: 20px 24px;
  background: white;
  border-radius: 16px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.08);

  .header-title {
    display: flex;
    align-items: center;

    span {
      font-size: 24px;
      font-weight: 800;
      background: linear-gradient(135deg, #6B4423 0%, #8D6E63 100%);
      -webkit-background-clip: text;
      -webkit-text-fill-color: transparent;
      background-clip: text;
    }
  }

  :deep(.el-button--primary) {
    background: linear-gradient(135deg, #8D6E63 0%, #6B4423 100%);
    border: none;
    box-shadow: 0 4px 12px rgba(107, 68, 35, 0.3);
    transition: all 0.3s;
    font-weight: 700;
    padding: 12px 24px;
    border-radius: 12px;

    &:hover {
      transform: translateY(-2px);
      box-shadow: 0 6px 16px rgba(107, 68, 35, 0.4);
    }
  }
}

// 桌台卡片网格
.tables-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
  gap: 20px;
}

// 桌台卡片
.table-card {
  border-radius: 16px;
  border: 2px solid #F5E6D3;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
  transition: all 0.3s;
  overflow: hidden;

  &:hover {
    transform: translateY(-4px);
    box-shadow: 0 8px 24px rgba(0, 0, 0, 0.12);
  }

  &.occupied {
    border-color: #FFB74D;
    background: linear-gradient(135deg, #FFF8F0 0%, #FFF3E0 100%);
  }

  :deep(.el-card__header) {
    background: linear-gradient(135deg, #FFF8F0 0%, white 100%);
    border-bottom: 2px solid #F5E6D3;
    padding: 16px 20px;
  }

  :deep(.el-card__body) {
    padding: 20px;
  }
}

.table-card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;

  .table-info {
    display: flex;
    align-items: center;
    gap: 12px;

    .table-number {
      font-size: 24px;
      font-weight: 800;
      color: #6B4423;
      background: linear-gradient(135deg, #8D6E63 0%, #6B4423 100%);
      padding: 4px 12px;
      border-radius: 8px;
      color: white;
    }

    .table-name {
      font-size: 16px;
      font-weight: 600;
      color: #6B4423;
    }
  }

  .table-actions {
    :deep(.el-button) {
      border: 2px solid #E0E0E0;
      transition: all 0.3s;

      &:hover {
        border-color: #8D6E63;
        background: #FFF8F0;
      }
    }
  }
}

.table-card-body {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.table-status {
  display: flex;
  justify-content: center;

  :deep(.el-tag) {
    font-size: 14px;
    padding: 8px 20px;
    border-radius: 12px;
    font-weight: 600;
  }

  :deep(.el-tag--success) {
    background: linear-gradient(135deg, #81C784 0%, #66BB6A 100%);
    border: none;
    color: white;
  }

  :deep(.el-tag--warning) {
    background: linear-gradient(135deg, #FFB74D 0%, #FFA726 100%);
    border: none;
    color: white;
  }
}

// 座位数调整
.capacity-control {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background: linear-gradient(135deg, #FFF8F0 0%, #F5E6D3 100%);
  border-radius: 12px;

  .capacity-label {
    display: flex;
    align-items: center;
    gap: 8px;
    font-weight: 600;
    color: #6B4423;
  }

  .capacity-adjuster {
    display: flex;
    align-items: center;
    gap: 12px;

    .capacity-number {
      font-size: 20px;
      font-weight: 800;
      color: #6B4423;
      min-width: 32px;
      text-align: center;
    }

    :deep(.el-button) {
      background: white;
      border: 2px solid #8D6E63;
      color: #8D6E63;

      &:hover:not(:disabled) {
        background: #8D6E63;
        color: white;
      }

      &:disabled {
        opacity: 0.4;
        cursor: not-allowed;
      }
    }
  }
}

// 当前订单
.current-orders {
  .orders-header {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 12px;
    font-weight: 600;
    color: #6B4423;
    font-size: 14px;
  }

  .orders-list {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  .batch-actions {
    margin-top: 12px;
    padding-top: 12px;
    border-top: 2px dashed #E0E0E0;
    display: flex;
    gap: 8px;
    justify-content: space-between;

    .print-btn {
      flex: 1;
      background: linear-gradient(135deg, #1976D2 0%, #1565C0 100%);
      border: none;
      font-weight: 600;
      color: white;
      box-shadow: 0 4px 12px rgba(25, 118, 210, 0.3);
      transition: all 0.3s;

      &:hover {
        transform: translateY(-2px);
        box-shadow: 0 6px 16px rgba(25, 118, 210, 0.4);
        background: linear-gradient(135deg, #2196F3 0%, #1976D2 100%);
      }

      &:active {
        transform: translateY(0);
      }
    }

    .clear-btn {
      flex: 1;
      background: linear-gradient(135deg, #D32F2F 0%, #C62828 100%);
      border: none;
      font-weight: 600;
      color: white;
      box-shadow: 0 4px 12px rgba(211, 47, 47, 0.3);
      transition: all 0.3s;

      &:hover {
        transform: translateY(-2px);
        box-shadow: 0 6px 16px rgba(211, 47, 47, 0.4);
        background: linear-gradient(135deg, #F44336 0%, #D32F2F 100%);
      }

      &:active {
        transform: translateY(0);
      }
    }
  }

  .order-item {
    padding: 12px;
    background: white;
    border-radius: 8px;
    border: 1px solid #E0E0E0;
    cursor: pointer;
    transition: all 0.3s;

    &:hover {
      border-color: #8D6E63;
      box-shadow: 0 2px 8px rgba(141, 110, 99, 0.2);
      transform: translateX(4px);
    }

    .order-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 8px;

      .order-number {
        font-weight: 600;
        color: #6B4423;
        font-size: 14px;
      }
    }

    .order-service-info {
      margin: 8px 0;
      padding: 6px 8px;
      background: #FFF3E0;
      border-left: 3px solid #FF9800;
      border-radius: 4px;
      font-size: 13px;

      .service-info-label {
        color: #E65100;
        font-weight: 600;
        margin-right: 4px;
      }

      .service-info-value {
        color: #5D4037;
      }
    }

    .order-items-preview {
      margin: 8px 0;
      padding: 8px;
      background: #FFF8F0;
      border-radius: 6px;

      .item-preview {
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 4px 0;
        font-size: 13px;

        &:not(:last-child) {
          border-bottom: 1px dashed #E0E0E0;
          margin-bottom: 4px;
          padding-bottom: 4px;
        }

        .item-name {
          flex: 1;
          color: #6B4423;
          font-weight: 500;
        }

        .item-qty {
          color: #8D6E63;
          margin-left: 8px;
        }
      }
    }

    .order-amount {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-top: 8px;
      padding-top: 8px;
      border-top: 1px solid #E0E0E0;

      .amount-label {
        font-size: 13px;
        color: #8D6E63;
      }

      .amount-value {
        font-size: 16px;
        font-weight: 700;
        color: #D84315;
      }
    }
  }
}

.no-orders {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  padding: 32px 16px;
  color: #BCAAA4;
  
  :deep(.el-icon) {
    font-size: 48px;
  }

  span {
    font-size: 14px;
  }
}

// 对话框样式
:deep(.el-dialog) {
  border-radius: 20px;
  overflow: hidden;

  .el-dialog__header {
    background: linear-gradient(135deg, #FFF8F0 0%, white 100%);
    border-bottom: 2px solid #F5E6D3;
    padding: 24px;

    .el-dialog__title {
      font-size: 20px;
      font-weight: 800;
      color: #6B4423;
    }
  }

  .el-dialog__body {
    padding: 24px;
  }

  .el-form-item__label {
    font-weight: 600;
    color: #6B4423;
  }

  .el-input__inner {
    border-radius: 10px;
    border-color: #E0E0E0;

    &:focus {
      border-color: #8D6E63;
    }
  }

  .el-input-number {
    border-radius: 10px;

    .el-input__inner {
      border-radius: 10px;
    }
  }

  .el-select {
    .el-input__inner {
      border-radius: 10px;
    }
  }

  .el-dialog__footer {
    padding: 20px 24px;
    border-top: 2px solid #F5F5F5;

    .el-button {
      padding: 10px 24px;
      border-radius: 10px;
      font-weight: 700;
      border: none;

      &.el-button--default {
        background: #E0E0E0;
        color: #616161;
      }

      &.el-button--primary {
        background: linear-gradient(135deg, #8D6E63 0%, #6B4423 100%);
        box-shadow: 0 4px 12px rgba(107, 68, 35, 0.3);
      }
    }
  }
}

// 二维码样式
#qrcode {
  padding: 20px;
  background: linear-gradient(135deg, #FFF8F0 0%, white 100%);
  border-radius: 16px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  display: inline-block;

  img {
    border-radius: 12px;
  }
}

// 订单详情
.order-detail {
  .detail-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 12px 0;

    .detail-label {
      font-weight: 600;
      color: #6B4423;
    }

    .detail-value {
      color: #616161;
    }
  }

  .order-items {
    h4 {
      margin-bottom: 12px;
      color: #6B4423;
      font-weight: 700;
    }

    .order-item-detail {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 10px 12px;
      background: #FFF8F0;
      border-radius: 8px;
      margin-bottom: 8px;

      .item-name {
        flex: 1;
        color: #6B4423;
        font-weight: 600;
      }

      .item-quantity {
        color: #8D6E63;
        margin: 0 12px;
        min-width: 40px;
      }

      .item-price {
        font-weight: 700;
        color: #D84315;
        min-width: 80px;
        text-align: right;
        margin-right: 8px;
      }

      .el-button {
        margin-left: 8px;
      }
    }
  }

  .order-total {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 16px;
    background: linear-gradient(135deg, #FFF8F0 0%, #F5E6D3 100%);
    border-radius: 12px;
    margin-bottom: 16px;

    .total-label {
      font-size: 18px;
      font-weight: 700;
      color: #6B4423;
    }

    .total-amount {
      font-size: 24px;
      font-weight: 800;
      background: linear-gradient(135deg, #D84315 0%, #BF360C 100%);
      -webkit-background-clip: text;
      -webkit-text-fill-color: transparent;
      background-clip: text;
    }
  }

  .order-actions {
    display: flex;
    gap: 12px;
    justify-content: flex-end;

    :deep(.el-button) {
      border-radius: 10px;
      font-weight: 600;
      padding: 10px 20px;
    }
  }
}

:deep(.el-dropdown-menu) {
  border-radius: 12px;
  border: none;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.15);
  padding: 8px;

  .el-dropdown-menu__item {
    border-radius: 8px;
    padding: 10px 16px;
    font-weight: 600;
    transition: all 0.3s;
    display: flex;
    align-items: center;
    gap: 8px;

    &:hover {
      background: linear-gradient(135deg, #FFF8F0 0%, #F5E6D3 100%);
      color: #6B4423;
    }
  }
}
</style>
