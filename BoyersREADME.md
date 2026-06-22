/**
 * Boyer's Truck — Google Sheets → n8n bridge  (SINGLE-WORKFLOW version)
 * Posts every event to ONE webhook with an {action} field: receive | commit | ship.
 *
 * ONE-TIME SETUP:
 *   1. Extensions ▸ Apps Script, paste this file.
 *   2. Run setup() once and approve the prompt (installs the onEdit trigger;
 *      a simple trigger can't call external URLs — an installable one can).
 *   3. Settings tab: set n8n_webhook_base to your n8n base URL (no trailing slash).
 */
 
var WEBHOOK_PATH = '/webhook/boyers-inventory';  // single endpoint
 
function setup() {
  ScriptApp.getProjectTriggers().forEach(function (t) {
    if (t.getHandlerFunction() === 'onSheetEdit') ScriptApp.deleteTrigger(t);
  });
  ScriptApp.newTrigger('onSheetEdit')
    .forSpreadsheet(SpreadsheetApp.getActive()).onEdit().create();
  SpreadsheetApp.getActive().toast('Bridge installed. Edits now sync to n8n.');
}
 
function getBase_() {
  var sh = SpreadsheetApp.getActive().getSheetByName('Settings');
  var v = sh.getDataRange().getValues();
  for (var i = 1; i < v.length; i++) {
    if (v[i][0] === 'n8n_webhook_base') return String(v[i][1] || '').replace(/\/+$/, '');
  }
  return '';
}
 
function header_(sheet, name) {
  return sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0].indexOf(name) + 1;
}
 
function rowObject_(sheet, row) {
  var headers = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
  var data = sheet.getRange(row, 1, 1, sheet.getLastColumn()).getValues()[0];
  var o = {};
  headers.forEach(function (h, i) { if (h) o[String(h)] = data[i]; });
  return o;
}
 
function post_(payload) {
  var base = getBase_();
  if (!base) { SpreadsheetApp.getActive().toast('Set n8n_webhook_base in Settings'); return; }
  UrlFetchApp.fetch(base + WEBHOOK_PATH, {
    method: 'post', contentType: 'application/json',
    payload: JSON.stringify(payload), muteHttpExceptions: true
  });
}
 
function processed_(notes) {
  notes = String(notes || '');
  return notes.indexOf('auto:') !== -1 || notes.indexOf('sent:') !== -1;
}
 
// True only when a cell actually has a value (treats 0 as filled, blank as not).
function filled_(v) { return v !== '' && v !== null && v !== undefined; }
 
// Look up retail_per_each for an item from the Inventory tab (for QBO pricing).
function lookupRetail_(itemNumber) {
  var sh = SpreadsheetApp.getActive().getSheetByName('Inventory');
  if (!sh) return '';
  var data = sh.getDataRange().getValues();
  var iItem = data[0].indexOf('item_number');
  var iRetail = data[0].indexOf('retail_per_each');
  if (iItem < 0 || iRetail < 0) return '';
  for (var r = 1; r < data.length; r++) {
    if (String(data[r][iItem]) === String(itemNumber)) return data[r][iRetail];
  }
  return '';
}
 
function onSheetEdit(e) {
  var sheet = e.range.getSheet(), name = sheet.getName(), row = e.range.getRow();
  if (row === 1) return;
  if (name !== 'Transactions' && name !== 'Commitments') return;
 
  // Serialize concurrent edits so a row can't be sent twice.
  var lock = LockService.getDocumentLock();
  try { lock.waitLock(15000); } catch (err) { return; }
 
  try {
    if (name === 'Transactions') {
      var t = rowObject_(sheet, row);
      if (String(t.type).toLowerCase() === 'receive' && t.item_number && filled_(t.qty) && filled_(t.unit_cost) && t.vendor && !processed_(t.notes)) {
        var txnId = t.txn_id || ('TX-' + Date.now());
        if (!t.txn_id) sheet.getRange(row, header_(sheet, 'txn_id')).setValue(txnId);
        // Stamp BEFORE posting and flush, so a second invocation is blocked.
        sheet.getRange(row, header_(sheet, 'notes')).setValue('sent: receive');
        SpreadsheetApp.flush();
        post_({
          action: 'receive', txn_id: txnId, item_number: t.item_number, description: t.description,
          qty: t.qty, unit_cost: t.unit_cost, vendor: t.vendor || '', reference: t.reference || '',
          retail: lookupRetail_(t.item_number), user: t.user || Session.getActiveUser().getEmail()
        });
      }
    } else if (name === 'Commitments') {
      var c = rowObject_(sheet, row);
      if (!c.item_number) return;
      var status = String(c.status || '').toLowerCase();
 
      if (status === 'ship' && String(c.notes || '').indexOf('sent: ship') === -1) {
        sheet.getRange(row, header_(sheet, 'notes')).setValue('sent: ship');
        SpreadsheetApp.flush();
        post_({
          action: 'ship', commit_id: c.commit_id, item_number: c.item_number, qty: c.qty,
          customer: c.customer, job: c.job, retail: lookupRetail_(c.item_number),
          user: Session.getActiveUser().getEmail()
        });
      } else if (status === 'open' && filled_(c.qty) && c.customer && !processed_(c.notes)) {
        var cid = c.commit_id || ('CM-' + Date.now());
        if (!c.commit_id) sheet.getRange(row, header_(sheet, 'commit_id')).setValue(cid);
        sheet.getRange(row, header_(sheet, 'notes')).setValue('sent: commit');
        SpreadsheetApp.flush();
        post_({
          action: 'commit', commit_id: cid, item_number: c.item_number, qty: c.qty,
          customer: c.customer, job: c.job, substatus: c.substatus, vehicle_vin: c.vehicle_vin,
          deposit_amount: c.deposit_amount, retail: lookupRetail_(c.item_number)
        });
      }
    }
  } finally {
    lock.releaseLock();
  }
}
