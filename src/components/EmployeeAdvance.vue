<template>
  <div class="employee-advance">
    <header class="panel-header">
      <h1 class="title">Employee Advance</h1>
      <span class="branch">HO/ Branch</span>
    </header>

    <div class="mode-select">
      <label class="radio-label">
        <input type="radio" v-model="mode" value="payment" />
        Advance Payment
      </label>
      <label class="radio-label">
        <input type="radio" v-model="mode" value="adjustment" />
        Advance Adjustment
      </label>
    </div>

    <div v-if="actionMessage.text" :class="['action-message', actionMessage.type]">
      {{ actionMessage.text }}
    </div>

    <div class="form-grid">
      <div class="field">
        <label>Advance Purpose</label>
        <div class="input-with-select">
          <input v-model="form.advancePurpose" :readonly="(mode === 'payment' && paymentViewMode) || (mode === 'adjustment' && adjustmentViewMode)" />
          <span class="dropdown-icon">▼</span>
        </div>
      </div>
      <div class="field">
        <label>EIN</label>
        <div class="input-with-select">
          <input v-model="form.ein" placeholder="" :readonly="(mode === 'payment' && paymentViewMode) || (mode === 'adjustment' && adjustmentViewMode)" />
          <span class="dropdown-icon">▼</span>
        </div>
      </div>
      <div class="field">
        <label>Employee Name</label>
        <input v-model="form.employeeName" class="readonly" readonly />
      </div>
      <div class="field">
        <label>{{ mode === 'payment' ? 'Advance Date (ERP Open)' : 'Advance Adjustment Date' }}</label>
        <input
          v-model="form.advanceDate"
          type="date"
          class="date-input"
          :readonly="(mode === 'payment' && paymentViewMode) || (mode === 'adjustment' && adjustmentViewMode)"
        />
        <span v-if="form.advanceDate" class="date-format-hint">(DD-MMM-YY: {{ formatDateDisplay(form.advanceDate) }})</span>
      </div>
      <template v-if="mode === 'payment'">
        <div class="field">
          <label>Payment Method</label>
          <div class="input-with-select">
            <input v-model="form.paymentMethod" :readonly="paymentViewMode" />
            <span class="dropdown-icon">▼</span>
          </div>
        </div>
        <div class="field">
          <label>Advance Amount</label>
          <input v-model="form.advanceAmount" type="text" :readonly="paymentViewMode" />
        </div>
        <div class="field">
          <label>Previous Advance Balance</label>
          <input v-model="form.previousAdvanceBalance" class="readonly" readonly />
        </div>
        <div class="field ref-field">
          <label>Reference PR/ Demand</label>
          <div class="ref-row">
            <input v-model="form.referencePR" :readonly="paymentViewMode" />
            <button type="button" class="btn btn-getdata" title="GetData" :disabled="paymentViewMode" @click="onGetData">GetData</button>
          </div>
          <p class="help-text">A screen will be popped-up on clicking 'GetData' where purchase requisitions/demand can be selected from the list.</p>
        </div>
        <div class="field full-width">
          <label>Remarks</label>
          <textarea v-model="form.remarks" rows="3" :readonly="paymentViewMode"></textarea>
        </div>
      </template>
    </div>

    <!-- Advance Adjustment: detail table and extras -->
    <template v-if="mode === 'adjustment'">
      <div class="adjustment-table-wrap">
        <table class="adjustment-table">
          <thead>
            <tr>
              <th class="col-check"></th>
              <th>1. Adv. Date</th>
              <th>2. Adv. Purpose</th>
              <th class="col-amount">3. Adv. Amt.</th>
              <th class="col-amount">4. Bill Amt. For Adj.</th>
              <th class="col-amount">5. Cash Receive/(Pay) Amt.</th>
              <th>6. Remaining Balance</th>
              <th>7. Particulars</th>
              <th>8. Reference</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(row, idx) in adjustmentRows" :key="idx">
              <td class="col-check">
                <input type="checkbox" v-model="row.selected" :disabled="adjustmentViewMode" />
              </td>
              <td>{{ row.advDate }}</td>
              <td>{{ row.advPurpose }}</td>
              <td class="col-amount">{{ row.advAmt }}</td>
              <td class="col-amount">{{ row.billAmtForAdj }}</td>
              <td class="col-amount">{{ row.cashReceivePay }}</td>
              <td>{{ row.remainingBalance || '–' }}</td>
              <td>{{ row.particulars }}</td>
              <td>{{ row.reference }}</td>
            </tr>
            <tr class="total-row">
              <td colspan="3"><strong>Total</strong></td>
              <td class="col-amount"><strong>{{ totals.advAmt }}</strong></td>
              <td class="col-amount"><strong>{{ totals.billAmtForAdj }}</strong></td>
              <td class="col-amount"><strong>{{ totals.cashReceivePay }}</strong></td>
              <td colspan="4"></td>
            </tr>
          </tbody>
        </table>
      </div>

      <div class="summary-row">
        <div class="field inline">
          <label>Total Bill Amount</label>
          <input :value="summary.totalBillAmount" class="summary-input readonly" readonly />
        </div>
        <div class="field inline">
          <label>Total Cash Amount</label>
          <div class="cash-label-wrap">
            <input :value="summary.totalCashAmount" class="summary-input readonly" readonly />
            <span class="suffix">to be Received/ (to be Paid)</span>
          </div>
        </div>
        <div class="field inline">
          <label>Total Adjusted Adv. Amt.</label>
          <input :value="summary.totalAdjustedAdvAmt" class="summary-input readonly" readonly />
        </div>
      </div>

      <div class="attach-section">
        <label class="attach-label">Attach Bill Document</label>
        <div class="attach-row">
          <input
            v-model="attachDocumentPath"
            class="attach-input"
            placeholder="Select file or enter document link"
            :readonly="adjustmentViewMode"
          />
          <input
            ref="fileInputRef"
            type="file"
            class="file-input-hidden"
            accept=".pdf,.jpg,.jpeg,.png,.doc,.docx"
            multiple
            @change="onFileSelected"
          />
          <button type="button" class="btn btn-orange btn-icon" title="Browse" :disabled="adjustmentViewMode" @click="onBrowseDocument">📁</button>
          <button type="button" class="btn btn-green btn-icon" title="Add" :disabled="adjustmentViewMode" @click="onAddDocument">+</button>
        </div>
        <table class="doc-list-table">
          <thead>
            <tr>
              <th>Doc SL</th>
              <th>Doc Link</th>
              <th class="col-action"></th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(doc, idx) in docList" :key="doc.id">
              <td>{{ String(idx + 1).padStart(2, '0') }}</td>
              <td><input v-model="doc.link" class="doc-link-input" placeholder="Document path or link" :readonly="adjustmentViewMode" /></td>
              <td class="col-action">
                <button v-if="!adjustmentViewMode" type="button" class="btn-remove" title="Remove" @click="removeDocument(idx)">×</button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </template>

    <div class="actions">
      <button type="button" class="btn btn-reverse" @click="onReverse">Reverse</button>
      <button type="button" class="btn btn-edit" @click="onEdit">Edit</button>
      <button type="button" class="btn btn-save" @click="onSave">Save</button>
    </div>

    <!-- GetData popup: select PR/Demand -->
    <Teleport to="body">
      <div v-if="showGetDataPopup" class="popup-overlay" @click.self="closeGetDataPopup">
        <div class="popup-box">
          <div class="popup-header">
            <h2 class="popup-title">Select Purchase Requisitions / Demand</h2>
            <button type="button" class="popup-close" aria-label="Close" @click="closeGetDataPopup">×</button>
          </div>
          <div class="popup-body">
            <table class="popup-table">
              <thead>
                <tr>
                  <th class="col-check"></th>
                  <th>PR / Demand No.</th>
                  <th>Date</th>
                  <th>Amount</th>
                  <th>Status</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in prDemandList" :key="item.id">
                  <td class="col-check">
                    <input type="checkbox" :value="item.id" v-model="selectedPRIds" />
                  </td>
                  <td>{{ item.refNo }}</td>
                  <td>{{ item.date }}</td>
                  <td>{{ item.amount }}</td>
                  <td>{{ item.status }}</td>
                </tr>
              </tbody>
            </table>
          </div>
          <div class="popup-footer">
            <button type="button" class="btn btn-cancel" @click="closeGetDataPopup">Cancel</button>
            <button type="button" class="btn btn-getdata" @click="confirmGetDataPopup">OK</button>
          </div>
        </div>
      </div>
    </Teleport>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const mode = ref('payment')
const paymentViewMode = ref(false)
const adjustmentViewMode = ref(false)
const actionMessage = ref({ text: '', type: 'success' })
let messageTimer = null

const DEFAULT_ADJUSTMENT_ROWS = [
  { selected: true, advDate: '23-Dec-25', advPurpose: 'Spot Purchase', advAmt: '5,000', billAmtForAdj: '4,000', cashReceivePay: '1,000', remainingBalance: '', particulars: 'Adv. Adj. with bill and remaining cash received', reference: 'PR0905, 0911, 0956' },
  { selected: true, advDate: '02-Feb-26', advPurpose: 'Spot Purchase', advAmt: '12,000', billAmtForAdj: '12,500', cashReceivePay: '(500)', remainingBalance: '', particulars: 'Adv. Adj. with bill and excess exp. Paid in cash', reference: 'PR0223, 0723, 0951' }
]

const form = ref({
  advancePurpose: 'Spot Purchase',
  ein: '',
  employeeName: '',
  advanceDate: '',
  paymentMethod: 'Cash',
  advanceAmount: '',
  previousAdvanceBalance: '',
  referencePR: '',
  remarks: ''
})

const adjustmentRows = ref([
  {
    selected: true,
    advDate: '23-Dec-25',
    advPurpose: 'Spot Purchase',
    advAmt: '5,000',
    billAmtForAdj: '4,000',
    cashReceivePay: '1,000',
    remainingBalance: '',
    particulars: 'Adv. Adj. with bill and remaining cash received',
    reference: 'PR0905, 0911, 0956'
  },
  {
    selected: true,
    advDate: '02-Feb-26',
    advPurpose: 'Spot Purchase',
    advAmt: '12,000',
    billAmtForAdj: '12,500',
    cashReceivePay: '(500)',
    remainingBalance: '',
    particulars: 'Adv. Adj. with bill and excess exp. Paid in cash',
    reference: 'PR0223, 0723, 0951'
  }
])

const totals = computed(() => {
  const advAmt = adjustmentRows.value.reduce((s, r) => s + parseNumber(r.advAmt), 0)
  const billAmtForAdj = adjustmentRows.value.reduce((s, r) => s + parseNumber(r.billAmtForAdj), 0)
  const cashReceivePay = adjustmentRows.value.reduce((s, r) => s + parseNumber(r.cashReceivePay), 0)
  return {
    advAmt: formatNumber(advAmt),
    billAmtForAdj: formatNumber(billAmtForAdj),
    cashReceivePay: formatNumber(cashReceivePay)
  }
})

const summary = computed(() => ({
  totalBillAmount: totals.value.billAmtForAdj,
  totalCashAmount: totals.value.cashReceivePay,
  totalAdjustedAdvAmt: totals.value.advAmt
}))

const attachDocumentPath = ref('')
const fileInputRef = ref(null)
let docIdCounter = 0
const docList = ref([
  { id: ++docIdCounter, link: '' },
  { id: ++docIdCounter, link: '' }
])

function onBrowseDocument() {
  fileInputRef.value?.click()
}

function onFileSelected(event) {
  const files = event.target.files
  if (!files?.length) return
  const names = Array.from(files).map((f) => f.name)
  attachDocumentPath.value = names.length === 1 ? names[0] : names.join(', ')
  event.target.value = ''
}

function onAddDocument() {
  const path = attachDocumentPath.value?.trim()
  docList.value.push({ id: ++docIdCounter, link: path || '' })
  attachDocumentPath.value = ''
}

function removeDocument(index) {
  docList.value.splice(index, 1)
}

const showGetDataPopup = ref(false)
const selectedPRIds = ref([])
const prDemandList = ref([
  { id: 'PR0905', refNo: 'PR0905', date: '20-Dec-25', amount: '2,500', status: 'Open' },
  { id: 'PR0911', refNo: 'PR0911', date: '21-Dec-25', amount: '1,800', status: 'Open' },
  { id: 'PR0956', refNo: 'PR0956', date: '22-Dec-25', amount: '3,200', status: 'Open' },
  { id: 'PR0223', refNo: 'PR0223', date: '01-Feb-26', amount: '4,000', status: 'Open' },
  { id: 'PR0723', refNo: 'PR0723', date: '02-Feb-26', amount: '5,500', status: 'Open' },
  { id: 'PR0951', refNo: 'PR0951', date: '03-Feb-26', amount: '2,800', status: 'Open' }
])

function parseNumber(s) {
  if (!s) return 0
  const n = parseFloat(String(s).replace(/[(),\s]/g, (m) => (m === '(' ? '-' : '')))
  return isNaN(n) ? 0 : n
}

function formatNumber(n) {
  if (n < 0) return '(' + Math.abs(n).toLocaleString('en-IN') + ')'
  return n.toLocaleString('en-IN')
}

const MONTH_NAMES = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec']

function formatDateDisplay(isoDate) {
  if (!isoDate) return ''
  const d = new Date(isoDate + 'T00:00:00')
  if (isNaN(d.getTime())) return isoDate
  const day = String(d.getDate()).padStart(2, '0')
  const month = MONTH_NAMES[d.getMonth()]
  const year = String(d.getFullYear()).slice(-2)
  return `${day}-${month}-${year}`
}

function showMessage(text, type = 'success') {
  actionMessage.value = { text, type }
  if (messageTimer) clearTimeout(messageTimer)
  messageTimer = setTimeout(() => {
    actionMessage.value = { text: '', type: 'success' }
  }, 4000)
}

function onSave() {
  if (mode.value === 'payment') {
    const { ein, advanceDate, advanceAmount } = form.value
    if (!ein?.trim()) {
      showMessage('Please enter EIN.', 'error')
      return
    }
    if (!advanceDate?.trim()) {
      showMessage('Please enter Advance Date.', 'error')
      return
    }
    if (!advanceAmount?.trim() || isNaN(parseNumber(advanceAmount))) {
      showMessage('Please enter a valid Advance Amount.', 'error')
      return
    }
    paymentViewMode.value = true
    showMessage('Advance payment saved successfully.')
  } else {
    if (!form.value.advanceDate?.trim()) {
      showMessage('Please enter Advance Adjustment Date.', 'error')
      return
    }
    if (!adjustmentRows.value.length) {
      showMessage('Please add at least one adjustment row.', 'error')
      return
    }
    adjustmentViewMode.value = true
    showMessage('Advance adjustment saved successfully.')
  }
}

function onEdit() {
  if (mode.value === 'payment') {
    paymentViewMode.value = false
    actionMessage.value = { text: '', type: 'success' }
  } else {
    adjustmentViewMode.value = false
    actionMessage.value = { text: '', type: 'success' }
  }
}

function resetAdjustmentData() {
  adjustmentRows.value = DEFAULT_ADJUSTMENT_ROWS.map((r) => ({ ...r }))
  docList.value = [
    { id: ++docIdCounter, link: '' },
    { id: ++docIdCounter, link: '' }
  ]
  attachDocumentPath.value = ''
}

function onReverse() {
  if (mode.value === 'payment') {
    if (!confirm('Are you sure you want to reverse this advance payment? This action cannot be undone.')) return
    form.value.advancePurpose = 'Spot Purchase'
    form.value.ein = ''
    form.value.employeeName = ''
    form.value.advanceDate = ''
    form.value.paymentMethod = 'Cash'
    form.value.advanceAmount = ''
    form.value.previousAdvanceBalance = ''
    form.value.referencePR = ''
    form.value.remarks = ''
    paymentViewMode.value = false
    showMessage('Advance payment reversed.')
  } else {
    if (!confirm('Are you sure you want to reverse this advance adjustment? This action cannot be undone.')) return
    resetAdjustmentData()
    form.value.advancePurpose = 'Spot Purchase'
    form.value.ein = ''
    form.value.employeeName = ''
    form.value.advanceDate = ''
    adjustmentViewMode.value = false
    showMessage('Advance adjustment reversed.')
  }
}

function onGetData() {
  selectedPRIds.value = []
  const existing = form.value.referencePR ? form.value.referencePR.split(/,\s*/) : []
  prDemandList.value.forEach((item) => {
    if (existing.includes(item.refNo)) selectedPRIds.value.push(item.id)
  })
  showGetDataPopup.value = true
}

function closeGetDataPopup() {
  showGetDataPopup.value = false
}

function confirmGetDataPopup() {
  const refs = prDemandList.value
    .filter((item) => selectedPRIds.value.includes(item.id))
    .map((item) => item.refNo)
  form.value.referencePR = refs.join(', ')
  closeGetDataPopup()
}
</script>

<style scoped>
.employee-advance {
  max-width: 1100px;
  margin: 0 auto;
  background: #fff;
  border: 1px solid #d0d0d0;
  border-radius: 6px;
  padding: 24px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

.action-message {
  padding: 10px 14px;
  margin-bottom: 16px;
  border-radius: 4px;
  font-size: 14px;
  font-weight: 500;
}

.action-message.success {
  background: #dcfce7;
  color: #166534;
  border: 1px solid #86efac;
}

.action-message.error {
  background: #fee2e2;
  color: #991b1b;
  border: 1px solid #fca5a5;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  padding-bottom: 12px;
  border-bottom: 1px solid #e0e0e0;
}

.title {
  margin: 0;
  font-size: 1.25rem;
  font-weight: 600;
  color: #1a1a1a;
}

.branch {
  color: #555;
  font-size: 0.95rem;
}

.mode-select {
  display: flex;
  gap: 24px;
  margin-bottom: 20px;
}

.radio-label {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  font-weight: 500;
}

.radio-label input {
  width: 18px;
  height: 18px;
  accent-color: #333;
}

.form-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px 20px;
  margin-bottom: 20px;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.field.full-width {
  grid-column: 1 / -1;
}

.field label {
  font-weight: 500;
  color: #444;
  font-size: 13px;
}

.field input,
.field textarea {
  padding: 8px 10px;
  border: 1px solid #bbb;
  border-radius: 4px;
  font-size: 14px;
}

.field input.readonly,
.field input:read-only,
.field textarea:read-only {
  background: #e8e8e8;
  color: #555;
  cursor: not-allowed;
}

.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.input-with-select {
  position: relative;
}

.input-with-select input {
  width: 100%;
  padding-right: 28px;
}

.dropdown-icon {
  position: absolute;
  right: 10px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 10px;
  color: #666;
  pointer-events: none;
}

.date-input {
  max-width: 12rem;
}

.date-format-hint {
  display: block;
  margin-top: 4px;
  font-size: 12px;
  color: #666;
}

.ref-field .ref-row {
  display: flex;
  gap: 10px;
  align-items: center;
}

.ref-field .ref-row input {
  flex: 1;
}

.help-text {
  margin: 6px 0 0;
  font-size: 12px;
  color: #666;
  line-height: 1.4;
}

.btn {
  padding: 8px 18px;
  border: 1px solid #999;
  border-radius: 4px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  background: #f5f5f5;
}

.btn-getdata {
  background: #2563eb;
  color: #fff;
  border-color: #2563eb;
  white-space: nowrap;
}

.btn-getdata:hover {
  background: #1d4ed8;
}

.adjustment-table-wrap {
  overflow-x: auto;
  margin-bottom: 20px;
  border: 1px solid #d0d0d0;
  border-radius: 4px;
}

.adjustment-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
}

.adjustment-table th,
.adjustment-table td {
  border: 1px solid #d0d0d0;
  padding: 8px 10px;
  text-align: left;
}

.adjustment-table th {
  background: #f5f5f5;
  font-weight: 600;
  color: #333;
}

.adjustment-table .col-check {
  width: 36px;
  text-align: center;
}

.adjustment-table .col-amount {
  width: 8rem;
  min-width: 8rem;
  text-align: right;
  white-space: nowrap;
  box-sizing: border-box;
}

.adjustment-table .total-row {
  background: #f9f9f9;
  font-weight: 600;
}

.adjustment-table .total-row td:first-child {
  text-align: left;
}

.summary-row {
  display: flex;
  flex-wrap: wrap;
  gap: 24px;
  margin-bottom: 20px;
}

.field.inline {
  min-width: 180px;
}

.summary-input {
  width: 140px;
  padding: 8px 10px;
  border: 1px solid #bbb;
  border-radius: 4px;
}

.cash-label-wrap {
  display: flex;
  align-items: center;
  gap: 8px;
}

.cash-label-wrap .suffix {
  font-size: 12px;
  color: #666;
}

.attach-section {
  margin-bottom: 20px;
}

.attach-label {
  display: block;
  font-weight: 500;
  margin-bottom: 8px;
  color: #444;
}

.attach-row {
  display: flex;
  gap: 8px;
  align-items: center;
  margin-bottom: 12px;
}

.attach-input {
  flex: 1;
  max-width: 400px;
  padding: 8px 10px;
  border: 1px solid #bbb;
  border-radius: 4px;
}

.file-input-hidden {
  position: absolute;
  width: 0;
  height: 0;
  opacity: 0;
  pointer-events: none;
}

.doc-list-table .col-action {
  width: 48px;
  text-align: center;
  vertical-align: middle;
}

.btn-remove {
  background: #dc2626;
  color: #fff;
  border: none;
  width: 28px;
  height: 28px;
  border-radius: 4px;
  font-size: 18px;
  line-height: 1;
  cursor: pointer;
  padding: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

.btn-remove:hover {
  background: #b91c1c;
}

.btn-icon {
  padding: 8px 14px;
  font-size: 16px;
}

.btn-orange {
  background: #ea580c;
  color: #fff;
  border-color: #ea580c;
}

.btn-green {
  background: #16a34a;
  color: #fff;
  border-color: #16a34a;
}

.doc-list-table {
  width: 100%;
  max-width: 500px;
  border-collapse: collapse;
  font-size: 13px;
}

.doc-list-table th,
.doc-list-table td {
  border: 1px solid #d0d0d0;
  padding: 6px 10px;
  text-align: left;
}

.doc-list-table th {
  background: #f5f5f5;
  font-weight: 600;
}

.doc-link-input {
  width: 100%;
  padding: 6px 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.actions {
  display: flex;
  gap: 12px;
  margin-top: 24px;
  padding-top: 16px;
  border-top: 1px solid #e0e0e0;
}

.btn-reverse {
  background: #ea580c;
  color: #fff;
  border-color: #ea580c;
}

.btn-edit {
  background: #eab308;
  color: #1a1a1a;
  border-color: #ca9a04;
}

.btn-save {
  background: #16a34a;
  color: #fff;
  border-color: #16a34a;
}

.actions .btn:hover {
  opacity: 0.92;
}

/* GetData popup */
.popup-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
  padding: 20px;
}

.popup-box {
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
  max-width: 640px;
  width: 100%;
  max-height: 90vh;
  display: flex;
  flex-direction: column;
}

.popup-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  border-bottom: 1px solid #e0e0e0;
  flex-shrink: 0;
}

.popup-title {
  margin: 0;
  font-size: 1.1rem;
  font-weight: 600;
  color: #1a1a1a;
}

.popup-close {
  background: none;
  border: none;
  font-size: 24px;
  line-height: 1;
  color: #666;
  cursor: pointer;
  padding: 0 4px;
}

.popup-close:hover {
  color: #333;
}

.popup-body {
  padding: 20px;
  overflow: auto;
  flex: 1;
}

.popup-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
}

.popup-table th,
.popup-table td {
  border: 1px solid #d0d0d0;
  padding: 8px 12px;
  text-align: left;
}

.popup-table th {
  background: #f5f5f5;
  font-weight: 600;
  color: #333;
}

.popup-table .col-check {
  width: 40px;
  text-align: center;
}

.popup-footer {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  padding: 16px 20px;
  border-top: 1px solid #e0e0e0;
  flex-shrink: 0;
}

.btn-cancel {
  background: #e5e5e5;
  color: #333;
  border-color: #bbb;
}
</style>
