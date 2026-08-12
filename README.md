// صحتي — تذكير الفحوصات والأدوية
const STORAGE_KEY = 'se7ti_app_v4';
let data = { exams: [], meds: [], savedMedNames: [] };

function loadData() {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (raw) data = JSON.parse(raw);
    renderAll();
}
function saveData() {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
    renderAll();
}
function showToast(msg) {
    const t = document.getElementById('toast');
    t.textContent = msg;
    t.classList.add('show');
    setTimeout(() => t.classList.remove('show'), 2500);
}

function showTab(id) {
    document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
    document.querySelectorAll('.section').forEach(s => s.classList.remove('active'));
    event.target.classList.add('active');
    document.getElementById(id).classList.add('active');
    renderAll();
}

function today() { return new Date().toISOString().split('T')[0]; }
function addDays(dateStr, days) {
    const d = new Date(dateStr);
    d.setDate(d.getDate() + days);
    return d.toISOString().split('T')[0];
}
function addMonths(dateStr, months) {
    const d = new Date(dateStr);
    d.setMonth(d.getMonth() + parseInt(months));
    return d.toISOString().split('T')[0];
}
function daysDiff(from, to) {
    return Math.ceil((new Date(to) - new Date(from)) / (1000 * 60 * 60 * 24));
}
function fmtDate(d) {
    return new Date(d).toLocaleDateString('ar-SA', { year: 'numeric', month: 'long', day: 'numeric' });
}

/* ================== EXAMS ================== */
function addExam() {
    const name = document.getElementById('examName').value.trim();
    const date = document.getElementById('examDate').value;
    const freq = document.getElementById('examFreq').value;
    const nextDate = document.getElementById('examNextDate').value;
    const rec = document.getElementById('examRec').value.trim();
    if (!name || !date) return showToast('❌ أكمل الحقول الإلزامية');
    data.exams.push({ id: Date.now(), name, date, freq, nextDate, rec });
    saveData();
    document.getElementById('examName').value = '';
    document.getElementById('examDate').value = '';
    document.getElementById('examNextDate').value = '';
    document.getElementById('examRec').value = '';
    showToast('✅ تم حفظ الفحص');
}
function deleteExam(id) {
    data.exams = data.exams.filter(e => e.id !== id);
    saveData();
    showToast('🗑️ تم الحذف');
}
function getExamNext(e) {
    return e.nextDate || addMonths(e.date, e.freq);
}

/* ================== MEDS ================== */
function addMed() {
    const name = document.getElementById('medName').value.trim();
    const boxes = parseInt(document.getElementById('medBoxes').value) || 1;
    const perBox = parseInt(document.getElementById('medPerBox').value);
    const dose = parseInt(document.getElementById('medDose').value);
    const start = document.getElementById('medStart').value || today();
    if (!name || !perBox || !dose) return showToast('❌ أكمل جميع الحقول');
    const totalQty = boxes * perBox;
    const days = Math.floor(totalQty / dose);
    data.meds.push({ id: Date.now(), name, boxes, perBox, totalQty, dose, start, days });
    if (!data.savedMedNames.includes(name)) data.savedMedNames.push(name);
    saveData();
    document.getElementById('medName').value = '';
    document.getElementById('medBoxes').value = '1';
    document.getElementById('medPerBox').value = '';
    document.getElementById('medDose').value = '';
    document.getElementById('medStart').value = '';
    hideAutocomplete();
    showToast('✅ تم حفظ الدواء');
}
function deleteMed(id) {
    data.meds = data.meds.filter(m => m.id !== id);
    saveData();
    showToast('🗑️ تم الحذف');
}

/* ================== AUTOCOMPLETE ================== */
function setupAutocomplete() {
    const input = document.getElementById('medName');
    const list = document.getElementById('autocompleteList');
    input.addEventListener('input', () => {
        const val = input.value.trim();
        if (!val) { hideAutocomplete(); return; }
        const matches = data.savedMedNames.filter(n => n.includes(val));
        if (!matches.length) { hideAutocomplete(); return; }
        list.innerHTML = matches.map(n => `<div class="autocomplete-item" onclick="selectMedName('${n.replace(/'/g, "\\'")}')">${n}</div>`).join('');
        list.classList.add('show');
    });
    input.addEventListener('blur', () => setTimeout(() => hideAutocomplete(), 200));
    input.addEventListener('keydown', (e) => {
        const items = list.querySelectorAll('.autocomplete-item');
        let active = list.querySelector('.autocomplete-item.active');
        if (e.key === 'ArrowDown') {
            e.preventDefault();
            if (!active) items[0]?.classList.add('active');
            else { active.classList.remove('active'); const next = active.nextElementSibling; if (next) next.classList.add('active'); else items[0]?.classList.add('active'); }
        } else if (e.key === 'ArrowUp') {
            e.preventDefault();
            if (active) { active.classList.remove('active'); const prev = active.previousElementSibling; if (prev) prev.classList.add('active'); else items[items.length - 1]?.classList.add('active'); }
        } else if (e.key === 'Enter' && active) { e.preventDefault(); active.click(); }
        else if (e.key === 'Escape') hideAutocomplete();
    });
}
function selectMedName(name) {
    document.getElementById('medName').value = name;
    hideAutocomplete();
}
function hideAutocomplete() {
    document.getElementById('autocompleteList').classList.remove('show');
}
function renderSavedMedChips() {
    const container = document.getElementById('savedMedsChips');
    if (!data.savedMedNames.length) { container.innerHTML = ''; return; }
    container.innerHTML = '<div class="hint" style="margin-bottom:4px;">أدوية مسجلة سابقاً (اضغط للاختيار):</div>' +
        data.savedMedNames.map(n => `<span class="saved-med-chip" onclick="selectMedName('${n.replace(/'/g, "\\'")}')">${n}</span>`).join('');
}

/* ================== RENDER ================== */
function renderAll() {
    document.getElementById('statExams').textContent = data.exams.length;
    document.getElementById('statMeds').textContent = data.meds.length;
    renderExams();
    renderMeds();
    renderAlerts();
    renderSavedMedChips();
}

function renderExams() {
    const container = document.getElementById('examsList');
    if (!data.exams.length) {
        container.innerHTML = '<div class="empty"><div class="empty-icon">🩺</div>لا توجد فحوصات مسجلة</div>';
        return;
    }
    container.innerHTML = data.exams.map(e => {
        const next = getExamNext(e);
        const remaining = daysDiff(today(), next);
        let badge = remaining <= 7 ? '<span class="badge badge-day">قريب</span>' : '<span class="badge badge-ok">بعيد</span>';
        let nextLabel = e.nextDate ? '📅 موعد محدد' : '⏳ الفحص القادم (حساب تلقائي)';
        let recHtml = e.rec ? `<div class="item-rec">📝 توصية الطبيب: ${e.rec}</div>` : '';
        return `<div class="item">
            <button class="delete-btn" onclick="deleteExam(${e.id})">×</button>
            <div class="item-header"><div class="item-name">${e.name} ${badge}</div></div>
            <div class="item-meta">
                📅 آخر فحص: ${fmtDate(e.date)}<br>
                🔄 التكرار: كل ${e.freq} أشهر<br>
                ${nextLabel}: ${fmtDate(next)} (${remaining <= 0 ? 'متأخر!' : remaining + ' يوم'})
            </div>
            ${recHtml}
        </div>`;
    }).join('');
}

function renderMeds() {
    const container = document.getElementById('medsList');
    if (!data.meds.length) {
        container.innerHTML = '<div class="empty"><div class="empty-icon">💊</div>لا توجد أدوية مسجلة</div>';
        return;
    }
    container.innerHTML = data.meds.map(m => {
        const end = addDays(m.start, m.days);
        const remaining = daysDiff(today(), end);
        let badge = remaining <= 2 ? '<span class="badge badge-day">ينفد!</span>' : remaining <= 7 ? '<span class="badge badge-week">أسبوع</span>' : '<span class="badge badge-ok">متوفر</span>';
        return `<div class="item">
            <button class="delete-btn" onclick="deleteMed(${m.id})">×</button>
            <div class="item-header"><div class="item-name">${m.name} ${badge}</div></div>
            <div class="item-meta">
                📦 العلب: ${m.boxes} علبة × ${m.perBox} حبة = <strong>${m.totalQty} حبة</strong><br>
                💊 الجرعة: ${m.dose} حبة/يوم | تكفي: <strong>${m.days} يوم</strong><br>
                📅 البدء: ${fmtDate(m.start)} | الانتهاء: ${fmtDate(end)}<br>
                ⏳ المتبقي: ${remaining <= 0 ? 'انتهى!' : remaining + ' يوم'}
            </div>
        </div>`;
    }).join('');
}

function renderAlerts() {
    const container = document.getElementById('alertsList');
    let alerts = [];

    data.exams.forEach(e => {
        const nextExam = getExamNext(e);
        const daysToExam = daysDiff(today(), nextExam);
        if (daysToExam <= 7 && daysToExam > 0) {
            let desc = `الفحص القادم بتاريخ ${fmtDate(nextExam)} — ${daysToExam} أيام متبقية. اذهب للطبيب لحجز الموعد وطلب الوصفة الجديدة`;
            if (e.rec) desc += `<br>📝 توصية الطبيب: ${e.rec}`;
            alerts.push({ icon: '🩺', title: `فحص ${e.name}`, desc, urgency: daysToExam <= 3 ? 'day' : 'week' });
        } else if (daysToExam <= 0) {
            let desc = `كان المقرر بتاريخ ${fmtDate(nextExam)} — يرجى حجز موعد فوراً`;
            if (e.rec) desc += `<br>📝 توصية الطبيب: ${e.rec}`;
            alerts.push({ icon: '⚠️', title: `فحص ${e.name} متأخر!`, desc, urgency: 'day' });
        }
    });

    data.meds.forEach(m => {
        const end = addDays(m.start, m.days);
        const daysLeft = daysDiff(today(), end);
        if (daysLeft <= 2 && daysLeft > 0) {
            alerts.push({ icon: '💊', title: `دواء ${m.name} ينفد قريباً!`, desc: `يتبقى ${daysLeft} ${daysLeft === 1 ? 'يوم' : 'أيام'} فقط — تحضر لشراء العبوة الجديدة فوراً`, urgency: 'day' });
        } else if (daysLeft <= 0) {
            alerts.push({ icon: '❌', title: `دواء ${m.name} انتهى!`, desc: `انتهى بتاريخ ${fmtDate(end)} — يرجى شراء العبوة الجديدة`, urgency: 'day' });
        }
    });

    if (data.meds.length > 1) {
        const activeMeds = data.meds.filter(m => daysDiff(today(), addDays(m.start, m.days)) > 0);
        if (activeMeds.length > 1) {
            const medDays = activeMeds.map(m => ({ name: m.name, daysLeft: daysDiff(today(), addDays(m.start, m.days)) }));
            const shortest = medDays.reduce((a, b) => a.daysLeft < b.daysLeft ? a : b);
            if (shortest.daysLeft <= 7 && shortest.daysLeft > 2) {
                const otherMeds = medDays.filter(m => m.name !== shortest.name && m.daysLeft > shortest.daysLeft);
                let desc = `دواء "${shortest.name}" هو الأقرب للانتهاء (${shortest.daysLeft} يوم). اذهب للطبيب لإجراء الفحص وطلب وصفة جديدة لجميع أدويتك.`;
                if (otherMeds.length > 0) desc += ` (باقي الأدوية: ${otherMeds.map(m => m.name + ' ' + m.daysLeft + ' يوم').join('، ')})`;
                alerts.push({ icon: '📋', title: 'تنبيه ذكي — جدولة الطبيب', desc, urgency: 'week' });
            }
        }
    }

    if (!alerts.length) {
        container.innerHTML = '<div class="empty"><div class="empty-icon">🎉</div>لا توجد تنبيهات حالياً — كل شيء تحت السيطرة!</div>';
        return;
    }
    alerts.sort((a, b) => (a.urgency === 'day' ? -1 : 1));
    container.innerHTML = alerts.map(a => `
        <div class="alert">
            <div class="alert-icon">${a.icon}</div>
            <div class="alert-text">
                <div class="alert-title">${a.title}</div>
                <div class="alert-desc">${a.desc}</div>
            </div>
            <span class="badge badge-${a.urgency}">${a.urgency === 'day' ? 'عاجل' : 'تنبيه'}</span>
        </div>
    `).join('');
}

document.addEventListener('DOMContentLoaded', () => {
    document.getElementById('examDate').value = today();
    document.getElementById('medStart').value = today();
    setupAutocomplete();
    loadData();
    if (data.exams.length || data.meds.length) {
        setTimeout(() => { const alerts = document.querySelectorAll('.alert').length; if (alerts > 0) showToast(`🔔 لديك ${alerts} تنبيهات جديدة`); }, 500);
    }
});
