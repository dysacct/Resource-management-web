<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { deleteMachines, deleteNetworks, getDeletedRecords, downloadExport } from '../api'

const activeTab = ref<'machine' | 'network'>('machine')
const loading = ref(false)
const machineIdcCode = ref('')
const machineIPMIs = ref('')
const networkIdcCode = ref('')
const networkIPMIs = ref('')
const showModal = ref(false)
const modalType = ref<'machine' | 'network'>('machine')
const modalPreview = ref<any[]>([])
const records = ref<any[]>([])
const recordsTotal = ref(0)
const recordsPage = ref(1)
const expandedGroup = ref<string | null>(null)

const groupedRecords = computed(() => {
  // 按 ipmi_ip 分组（同一台机器可能有3条记录）
  const groups: Record<string, any[]> = {}
  for (const rec of records.value) {
    const key = rec.ipmi_ip || `rec-${rec.id}`
    if (!groups[key]) groups[key] = []
    groups[key].push(rec)
  }
  return Object.entries(groups).map(([ipmi, recs]) => ({
    ipmi,
    records: recs,
    deletedAt: recs[0].deleted_at,
    deletedBy: recs[0].deleted_by,
    expiresAt: recs[0].expires_at,
    idcCode: recs[0].idc_code || '-',
  }))
})

onMounted(() => fetchRecords())

async function fetchRecords() {
  try {
    const res = await getDeletedRecords({ page: String(recordsPage.value), size: '100', record_type: activeTab.value })
    if (res.code === 200) {
      records.value = res.data?.list || []
      recordsTotal.value = res.data?.total || 0
    }
  } catch {}
}

function openModal(type: 'machine' | 'network') {
  modalType.value = type
  if (type === 'machine') {
    const idc = machineIdcCode.value.trim()
    const ips = machineIPMIs.value.split(/[\n,]/).map(s => s.trim()).filter(s => s.length > 0)
    if (!idc && ips.length === 0) { alert('请填写机房编码或 IPMI IP'); return }
    modalPreview.value = idc ? [{ label: '机房编码', value: idc }] : ips.map(ip => ({ label: 'IPMI IP', value: ip }))
  } else {
    const idc = networkIdcCode.value.trim()
    const ips = networkIPMIs.value.split(/[\n,]/).map(s => s.trim()).filter(s => s.length > 0)
    if (!idc && ips.length === 0) { alert('请填写机房编码或 IPMI IP'); return }
    modalPreview.value = idc ? [{ label: '机房编码', value: idc }] : ips.map(ip => ({ label: 'IPMI IP', value: ip }))
  }
  showModal.value = true
}

async function confirmDelete() {
  showModal.value = false
  loading.value = true
  try {
    if (modalType.value === 'machine') {
      const idc = machineIdcCode.value.trim()
      const ips = machineIPMIs.value.split(/[\n,]/).map(s => s.trim()).filter(s => s.length > 0)
      const res = await deleteMachines({ idc_code: idc || undefined, ipmi_ips: ips.length > 0 ? ips : undefined })
      if (res.code === 200) { alert(res.message); machineIdcCode.value = ''; machineIPMIs.value = ''; fetchRecords() }
      else { alert('删除失败: ' + res.message) }
    } else {
      const idc = networkIdcCode.value.trim()
      const ips = networkIPMIs.value.split(/[\n,]/).map(s => s.trim()).filter(s => s.length > 0)
      const res = await deleteNetworks({ idc_code: idc || undefined, ipmi_ips: ips.length > 0 ? ips : undefined })
      if (res.code === 200) { alert(res.message); networkIdcCode.value = ''; networkIPMIs.value = ''; fetchRecords() }
      else { alert('删除失败: ' + res.message) }
    }
  } catch (e: any) { alert('请求失败: ' + e.message) }
  finally { loading.value = false }
}

function switchTab(tab: 'machine' | 'network') {
  activeTab.value = tab
  expandedGroup.value = null
  recordsPage.value = 1
  fetchRecords()
}

function daysLeft(expiresAt: string): number {
  return Math.max(0, Math.ceil((new Date(expiresAt).getTime() - Date.now()) / (1000 * 60 * 60 * 24)))
}

function parseData(data: string): any {
  try { return JSON.parse(data) } catch { return {} }
}

function toggleGroup(ipmi: string) {
  expandedGroup.value = expandedGroup.value === ipmi ? null : ipmi
}

function doExport() {
  downloadExport('/deletion/records/export', { record_type: activeTab.value })
}
</script>

<template>
  <div class="page-header">
    <h2>删除管理</h2>
    <p class="page-hint">删除的数据将归档保留30天，到期后自动清理。删除操作不可撤销，请谨慎操作。</p>
  </div>

  <div class="deletion-panels">
    <div class="deletion-card" :class="{ active: activeTab === 'machine' }" @click="switchTab('machine')">
      <h3>删除机器信息（三表）</h3>
      <p>删除 idc_info + machine_info + business_info</p>
      <div v-if="activeTab === 'machine'" class="deletion-form" @click.stop>
        <div class="form-group">
          <label>按机房编码删除</label>
          <input v-model="machineIdcCode" placeholder="如 B11" :disabled="!!machineIPMIs.trim()" />
        </div>
        <div class="form-group">
          <label>按 IPMI IP 删除（多个用逗号或换行分隔）</label>
          <textarea v-model="machineIPMIs" placeholder="10.0.0.1&#10;10.0.0.2" rows="4" :disabled="!!machineIdcCode.trim()"></textarea>
        </div>
        <button class="btn-danger" :disabled="loading" @click="openModal('machine')">{{ loading ? '删除中...' : '删除机器信息' }}</button>
      </div>
    </div>

    <div class="deletion-card" :class="{ active: activeTab === 'network' }" @click="switchTab('network')">
      <h3>删除网络信息（单表）</h3>
      <p>仅删除 network_info 表中的记录</p>
      <div v-if="activeTab === 'network'" class="deletion-form" @click.stop>
        <div class="form-group">
          <label>按机房编码删除</label>
          <input v-model="networkIdcCode" placeholder="如 B11" :disabled="!!networkIPMIs.trim()" />
        </div>
        <div class="form-group">
          <label>按 IPMI IP 删除</label>
          <textarea v-model="networkIPMIs" placeholder="10.0.0.1&#10;10.0.0.2" rows="4" :disabled="!!networkIdcCode.trim()"></textarea>
        </div>
        <button class="btn-danger" :disabled="loading" @click="openModal('network')">{{ loading ? '删除中...' : '删除网络信息' }}</button>
      </div>
    </div>
  </div>

  <!-- 确认弹框 -->
  <div v-if="showModal" class="modal-overlay" @click.self="showModal = false">
    <div class="modal-box">
      <div class="modal-header"><h3>确认删除</h3></div>
      <div class="modal-body">
        <p class="modal-warn">此操作不可撤销！将删除以下内容并归档30天：</p>
        <div class="modal-list">
          <div v-for="(item, idx) in modalPreview" :key="idx" class="modal-item">
            <span class="modal-label">{{ item.label }}：</span><code>{{ item.value }}</code>
          </div>
        </div>
        <p class="modal-type">删除类型：<strong>{{ modalType === 'machine' ? '机器信息（三表）' : '网络信息（单表）' }}</strong></p>
      </div>
      <div class="modal-footer">
        <button class="btn-cancel" @click="showModal = false">取消</button>
        <button class="btn-danger" :disabled="loading" @click="confirmDelete">{{ loading ? '删除中...' : '确认删除' }}</button>
      </div>
    </div>
  </div>

  <!-- 删除记录列表 -->
  <div class="data-card" style="margin-top:20px">
    <div class="table-header-bar">
      <h3 style="margin:0;font-size:15px">删除记录（保留30天，{{ activeTab === 'machine' ? '机器' : '网络' }}，按IP分组）</h3>
      <div style="display:flex;gap:8px;align-items:center">
        <span style="font-size:13px;color:#999">共 {{ groupedRecords.length }} 组</span>
        <button style="background:#52c41a;color:#fff;border:none;padding:6px 14px;border-radius:4px;font-size:12px;cursor:pointer" @click="doExport">导出Excel</button>
      </div>
    </div>

    <div v-if="groupedRecords.length === 0" class="empty" style="padding:30px">暂无删除记录</div>
    <template v-else>
      <div class="table-wrapper">
        <table>
          <thead>
            <tr>
              <th style="width:36px"></th>
              <th>IPMI IP</th>
              <th>机房</th>
              <th>记录数</th>
              <th>操作人</th>
              <th>删除时间</th>
              <th>剩余天数</th>
            </tr>
          </thead>
          <tbody>
            <template v-for="grp in groupedRecords" :key="grp.ipmi">
              <tr @click="toggleGroup(grp.ipmi)" style="cursor:pointer">
                <td><span style="color:#999;font-size:11px">{{ expandedGroup === grp.ipmi ? '▼' : '▶' }}</span></td>
                <td><code>{{ grp.ipmi }}</code></td>
                <td>{{ grp.idcCode }}</td>
                <td>{{ grp.records.length }}</td>
                <td>{{ grp.deletedBy }}</td>
                <td>{{ new Date(grp.deletedAt).toLocaleString('zh-CN') }}</td>
                <td><span :class="daysLeft(grp.expiresAt) <= 3 ? 'days-warn' : 'days-ok'">{{ daysLeft(grp.expiresAt) }} 天</span></td>
              </tr>
              <tr v-if="expandedGroup === grp.ipmi" class="detail-row">
                <td colspan="7">
                  <div class="detail-panel">
                    <div v-for="rec in grp.records" :key="rec.id" class="rec-detail">
                      <h4 class="rec-title">{{ rec.source_table }}</h4>
                      <template v-if="rec.source_table === 'idc_info'">
                        <div class="detail-grid">
                          <div class="detail-item"><strong>ZbxID:</strong>{{ parseData(rec.record_data).zbx_id }}</div>
                          <div class="detail-item"><strong>IPMI IP:</strong>{{ parseData(rec.record_data).ipmi_ip }}</div>
                          <div class="detail-item"><strong>机房编码:</strong>{{ parseData(rec.record_data).idc_code }}</div>
                          <div class="detail-item"><strong>机房名称:</strong>{{ parseData(rec.record_data).idc_name }}</div>
                          <div class="detail-item"><strong>SSH IP:</strong>{{ parseData(rec.record_data).ssh_ip }}</div>
                        </div>
                      </template>
                      <template v-else-if="rec.source_table === 'machine_info'">
                        <div class="detail-grid">
                          <div class="detail-item"><strong>系统:</strong>{{ parseData(rec.record_data).system_type }}</div>
                          <div class="detail-item"><strong>厂商:</strong>{{ parseData(rec.record_data).manufacturer }}</div>
                          <div class="detail-item"><strong>序列号:</strong>{{ parseData(rec.record_data).server_sn }}</div>
                          <div class="detail-item"><strong>系统盘:</strong>{{ parseData(rec.record_data).system_disk }}</div>
                          <div class="detail-item"><strong>SSD:</strong>{{ parseData(rec.record_data).ssd_count }}</div>
                          <div class="detail-item"><strong>HDD:</strong>{{ parseData(rec.record_data).hdd_count }}</div>
                          <div class="detail-item"><strong>内存:</strong>{{ parseData(rec.record_data).memory_count }}</div>
                          <div class="detail-item"><strong>CPU:</strong>{{ parseData(rec.record_data).cpu_info }}</div>
                          <div class="detail-item"><strong>高度:</strong>{{ parseData(rec.record_data).server_height }}</div>
                        </div>
                      </template>
                      <template v-else-if="rec.source_table === 'business_info'">
                        <div class="detail-grid">
                          <div class="detail-item"><strong>业务名:</strong>{{ parseData(rec.record_data).business_name }}</div>
                          <div class="detail-item"><strong>业务ID:</strong>{{ parseData(rec.record_data).business_id }}</div>
                          <div class="detail-item"><strong>带宽:</strong>{{ parseData(rec.record_data).business_speed }}M</div>
                          <div class="detail-item"><strong>旧业务:</strong>{{ parseData(rec.record_data).old_business_name }}</div>
                          <div class="detail-item"><strong>旧业务ID:</strong>{{ parseData(rec.record_data).old_business_id }}</div>
                          <div class="detail-item"><strong>旧带宽:</strong>{{ parseData(rec.record_data).old_business_speed }}M</div>
                        </div>
                      </template>
                      <template v-else-if="rec.source_table === 'network_info'">
                        <div class="detail-grid">
                          <div class="detail-item"><strong>IPv4:</strong>{{ parseData(rec.record_data).ipv4_ip }}</div>
                          <div class="detail-item"><strong>IPv6:</strong>{{ parseData(rec.record_data).ipv6_ip }}</div>
                          <div class="detail-item"><strong>MAC:</strong>{{ parseData(rec.record_data).mac_address }}</div>
                          <div class="detail-item"><strong>网卡:</strong>{{ parseData(rec.record_data).eth_name }}</div>
                          <div class="detail-item"><strong>机房:</strong>{{ parseData(rec.record_data).idc_code }}</div>
                          <div class="detail-item"><strong>网络类型:</strong>{{ parseData(rec.record_data).net_type }}</div>
                          <div class="detail-item"><strong>VLAN:</strong>{{ parseData(rec.record_data).vlan }}</div>
                          <div class="detail-item"><strong>网关:</strong>{{ parseData(rec.record_data).ipv4_gateway }}</div>
                          <div class="detail-item"><strong>速率:</strong>{{ parseData(rec.record_data).ip_speed }}</div>
                          <div class="detail-item"><strong>状态:</strong>{{ parseData(rec.record_data).ip_status }}</div>
                          <div class="detail-item"><strong>备注:</strong>{{ parseData(rec.record_data).ip_notes }}</div>
                        </div>
                      </template>
                    </div>
                  </div>
                </td>
              </tr>
            </template>
          </tbody>
        </table>
      </div>
    </template>
  </div>
</template>

<style scoped>
.deletion-panels { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
.deletion-card { background: #fff; border: 2px solid #e8e8e8; border-radius: 10px; padding: 24px; cursor: pointer; transition: border-color 0.2s; }
.deletion-card:hover { border-color: #d0d0d0; }
.deletion-card.active { border-color: #ff4d4f; }
.deletion-card h3 { font-size: 16px; margin: 0 0 4px; color: #333; }
.deletion-card > p { font-size: 13px; color: #999; margin: 0 0 12px; }
.deletion-form { margin-top: 16px; }
.deletion-form .form-group { margin-bottom: 14px; }
.deletion-form label { display: block; font-size: 13px; color: #666; margin-bottom: 4px; }
.deletion-form input, .deletion-form textarea { width: 100%; padding: 8px 12px; border: 1px solid #d9d9d9; border-radius: 6px; font-size: 13px; font-family: inherit; }
.deletion-form input:focus, .deletion-form textarea:focus { outline: none; border-color: #ff4d4f; }
.btn-danger { padding: 10px 24px; background: #ff4d4f; color: #fff; border: none; border-radius: 6px; font-size: 14px; cursor: pointer; }
.btn-danger:hover:not(:disabled) { background: #e63946; }
.btn-danger:disabled { opacity: 0.5; cursor: not-allowed; }

.modal-overlay { position: fixed; inset: 0; background: rgba(0,0,0,0.45); display: flex; align-items: center; justify-content: center; z-index: 2000; }
.modal-box { background: #fff; border-radius: 12px; width: 480px; max-width: 90vw; box-shadow: 0 8px 40px rgba(0,0,0,0.2); }
.modal-header { padding: 20px 24px 0; }
.modal-header h3 { font-size: 18px; }
.modal-body { padding: 16px 24px; }
.modal-warn { color: #ff4d4f; font-size: 14px; margin: 0 0 16px; }
.modal-list { background: #fafafa; border-radius: 8px; padding: 12px 16px; margin-bottom: 16px; max-height: 200px; overflow-y: auto; }
.modal-item { padding: 6px 0; font-size: 14px; border-bottom: 1px solid #f0f0f0; }
.modal-item:last-child { border-bottom: none; }
.modal-label { color: #999; }
.modal-item code { background: #fff; padding: 2px 8px; border-radius: 4px; font-size: 13px; color: #333; }
.modal-type { font-size: 13px; color: #666; }
.modal-footer { display: flex; justify-content: flex-end; gap: 12px; padding: 16px 24px; border-top: 1px solid #f0f0f0; }
.btn-cancel { padding: 10px 24px; background: #fff; color: #666; border: 1px solid #d9d9d9; border-radius: 6px; font-size: 14px; cursor: pointer; }
.btn-cancel:hover { border-color: #999; }

.table-header-bar { display: flex; justify-content: space-between; align-items: center; padding: 14px 20px; border-bottom: 1px solid #f0f0f0; }
.days-ok { color: #52c41a; font-weight: 600; }
.days-warn { color: #ff4d4f; font-weight: 600; }
.detail-row td { padding: 0 !important; }
.detail-panel { background: #fafafa; padding: 14px 20px; }
.rec-detail { margin-bottom: 12px; padding-bottom: 12px; border-bottom: 1px dashed #e8e8e8; }
.rec-detail:last-child { margin-bottom: 0; padding-bottom: 0; border-bottom: none; }
.rec-title { font-size: 13px; color: #d4724a; margin: 0 0 8px; }
.detail-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 4px 20px; }
.detail-item { font-size: 12px; color: #666; }
.detail-item strong { color: #333; margin-right: 4px; }

@media (max-width: 768px) { .deletion-panels { grid-template-columns: 1fr; } }
</style>
