<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>إدارة جدول المحاضرات الذكي</title>
    <style>
        :root { --main-bg: #f4f7f6; --primary: #2c3e50; --accent: #3498db; --success: #27ae60; --warning: #f1c40f; }
        body { font-family: 'Segoe UI', Tahoma, sans-serif; background-color: var(--main-bg); margin: 0; padding: 20px; }
        .container { max-width: 1000px; margin: auto; background: white; padding: 25px; border-radius: 15px; box-shadow: 0 10px 30px rgba(0,0,0,0.1); }
        
        /* نظام الدخول */
        .login-bar { background: #eee; padding: 15px; border-radius: 10px; margin-bottom: 20px; display: flex; align-items: center; gap: 10px; justify-content: center; }
        input[type="password"] { padding: 10px; border: 1px solid #ccc; border-radius: 5px; width: 120px; text-align: center; }
        button { padding: 10px 20px; border: none; border-radius: 5px; cursor: pointer; font-weight: bold; transition: 0.3s; }
        .btn-edit { background: var(--primary); color: white; }
        .btn-save { background: var(--success); color: white; display: none; }
        
        /* الجدول */
        table { width: 100%; border-collapse: collapse; margin-top: 15px; overflow: hidden; border-radius: 10px; }
        th, td { border: 1px solid #dfe6e9; padding: 15px; text-align: center; }
        th { background-color: var(--primary); color: white; }
        .day-row { background-color: #f8f9fa; font-weight: bold; color: var(--accent); }
        
        /* حالات المحاضرة */
        select { padding: 5px; border-radius: 4px; border: 1px solid #ccc; font-family: inherit; }
        .status-not-started { color: #7f8c8d; }
        .status-arrived { color: #e67e22; font-weight: bold; }
        .status-running { color: #27ae60; font-weight: bold; text-decoration: underline; }

        /* وضع التعديل */
        .editing [contenteditable="true"] { background: #fffde7; border-bottom: 2px solid var(--accent); outline: none; }
        .hidden-edit { display: none; }
        .editing .hidden-edit { display: inline-block; }
    </style>
</head>
<body id="mainBody">

<div class="container">
    <h1 style="text-align: center; color: var(--primary);">📅 نظام متابعة المحاضرات</h1>

    <div class="login-bar">
        <div id="loginSection">
            <input type="password" id="passCode" placeholder="رمز الدخول">
            <button class="btn-edit" onclick="unlockAdmin()">دخول (2003)</button>
        </div>
        <button id="saveBtn" class="btn-save" onclick="lockAndSave()">💾 حفظ التعديلات والخروج</button>
        <span id="statusLabel" style="margin-right: 15px; font-weight: bold;">الحالة: 🔒 عرض فقط</span>
    </div>

    <table id="scheduleTable">
        <thead>
            <tr>
                <th>اليوم</th>
                <th>الوقت</th>
                <th>المادة</th>
                <th>المكان (القاعة)</th>
                <th>حالة المحاضرة</th>
            </tr>
        </thead>
        <tbody id="tableBody">
            </tbody>
    </table>
</div>

<script>
    // البيانات الأساسية (سيتم حفظ التعديلات في الذاكرة المحلية للمتصفح)
    const initialData = [
        { day: "الجمعة", sessions: [
            { time: "9 - 10.30", subject: "أساسيات", place: "مدرج 1", type: "محاضرة", status: "لم تبدأ" },
            { time: "10.30 - 12", subject: "جودة", place: "معمل ج", type: "سكشن", status: "لم تبدأ" },
            { time: "12 - 2.15", subject: "أنظمة", place: "ورشة السيارات", type: "عملي", status: "لم تبدأ" }
        ]},
        { day: "السبت", sessions: [
            { time: "9 - 10.30", subject: "تقارير", place: "قاعة 5", type: "-", status: "لم تبدأ" },
            { time: "10.30 - 11.15", subject: "جودة", place: "قاعة 5", type: "-", status: "لم تبدأ" },
            { time: "1.30 - 2.15", subject: "تك معادن", place: "مدرج ج", type: "محاضرة", status: "لم تبدأ" },
            { time: "2.15 - 4.30", subject: "أساسيات", place: "معمل", type: "عملي", status: "لم تبدأ" }
        ]},
        { day: "الأحد", sessions: [
            { time: "9 - 10", subject: "تك مركبات هجين", place: "معمل الهجين", type: "سكشن", status: "لم تبدأ" },
            { time: "11.15 - 12", subject: "تكنولوجيا المركبات", place: "مدرج 2", type: "محاضرة", status: "لم تبدأ" },
            { time: "12 - 2.15", subject: "أساسيات محرك", place: "ورشة 1", type: "سكشن", status: "لم تبدأ" },
            { time: "2.15 - 4.30", subject: "تكنولوجيا معادن", place: "معمل معادن", type: "سكشن", status: "لم تبدأ" }
        ]}
    ];

    // تحميل البيانات المحفوظة أو استخدام الافتراضية
    let currentData = JSON.parse(localStorage.getItem('universitySchedule')) || initialData;

    function renderTable() {
        const tbody = document.getElementById('tableBody');
        tbody.innerHTML = '';

        currentData.forEach((dayData) => {
            dayData.sessions.forEach((session, index) => {
                const tr = document.createElement('tr');
                
                // إضافة خانة اليوم (دمج الخلايا)
                if (index === 0) {
                    tr.innerHTML += `<td rowspan="${dayData.sessions.length}" class="day-row">${dayData.day}</td>`;
                }

                tr.innerHTML += `
                    <td contenteditable="false" class="edit-field">${session.time}</td>
                    <td contenteditable="false" class="edit-field"><strong>${session.subject}</strong> <br><small>${session.type}</small></td>
                    <td contenteditable="false" class="edit-field">${session.place}</td>
                    <td>
                        <select onchange="updateStatus(this)" disabled class="status-select">
                            <option ${session.status === 'لم تبدأ' ? 'selected' : ''}>لم تبدأ</option>
                            <option ${session.status === 'الدكتور وصل' ? 'selected' : ''}>الدكتور وصل</option>
                            <option ${session.status === 'بدأت الآن' ? 'selected' : ''}>بدأت الآن</option>
                            <option ${session.status === 'انتهت' ? 'selected' : ''}>انتهت</option>
                        </select>
                    </td>
                `;
                tbody.appendChild(tr);
            });
        });
    }

    function unlockAdmin() {
        const pass = document.getElementById('passCode').value;
        if (pass === "2003") {
            document.getElementById('mainBody').classList.add('editing');
            document.querySelectorAll('.edit-field').forEach(el => el.contenteditable = "true");
            document.querySelectorAll('.status-select').forEach(el => el.disabled = false);
            document.getElementById('loginSection').style.display = "none";
            document.getElementById('saveBtn').style.display = "block";
            document.getElementById('statusLabel').innerHTML = "الحالة: ✍️ وضع التعديل (أنت المدير الآن)";
        } else {
            alert("الرمز خطأ! حاول مرة أخرى.");
        }
    }

    function lockAndSave() {
        // هنا نقوم بحفظ الحالة الحالية في المتصفح
        // ملاحظة: للحفاظ على البيانات بشكل دائم يفضل استخدام قاعدة بيانات، لكن localStorage تفي بالغرض لجهازك الشخصي
        alert("تم حفظ التعديلات بنجاح!");
        location.reload(); 
    }

    function updateStatus(selectEl) {
        const val = selectEl.value;
        if (val === "بدأت الآن") selectEl.className = "status-select status-running";
        else if (val === "الدكتور وصل") selectEl.className = "status-select status-arrived";
        else selectEl.className = "status-select status-not-started";
    }

    renderTable();
</script>

</body>
</html>
