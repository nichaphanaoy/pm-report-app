<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Preventive Maintenance Report</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Sarabun:wght@400;500;700&display=swap" rel="stylesheet">
    <style>
        body { font-family: 'Sarabun', sans-serif; }
        .view { transition: opacity 0.2s ease-in-out; }
        .hidden-view { display: none; }
        .status-button {
            border: 1px solid #e5e7eb;
            background-color: #f9fafb;
            color: #4b5563;
        }
        .status-button-active-pass { 
            background-color: #28a745; 
            color: white; 
            border-color: #28a745;
        }
        .status-button-active-fail { 
            background-color: #dc3545; 
            color: white; 
            border-color: #dc3545;
        }
    </style>
</head>
<body class="bg-gray-100 text-gray-800">

    <!-- Loading Spinner -->
    <div id="loading-spinner" class="fixed inset-0 bg-white/90 flex items-center justify-center z-50">
        <div class="flex flex-col items-center gap-4">
            <div class="animate-spin rounded-full h-12 w-12 border-t-4 border-b-4 border-blue-500"></div>
            <p id="loading-text" class="text-gray-600 font-medium">กำลังโหลด...</p>
        </div>
    </div>
    
    <!-- Header -->
    <header class="bg-white sticky top-0 z-40 border-b border-gray-200">
        <div class="container mx-auto px-4 sm:px-6 lg:px-8">
            <nav class="flex h-16">
                <button id="tab-form" onclick="switchView('form-view')" class="py-2 px-4 text-base font-semibold border-b-4 border-blue-500 text-blue-600">กรอกรายงาน</button>
                <button id="tab-history" onclick="switchView('history-view')" class="py-2 px-4 text-base font-semibold border-b-4 border-transparent text-gray-500 hover:border-gray-300 hover:text-gray-700">ประวัติการส่งรายงาน</button>
            </nav>
        </div>
    </header>

    <main class="container mx-auto p-4 sm:p-6 lg:p-8">
        <!-- ===== Main Form View ===== -->
        <div id="form-view" class="view">
            <div class="bg-white p-6 sm:p-8 rounded-xl shadow-sm mb-8 border border-gray-200">
                <h2 class="text-xl font-bold text-gray-800 mb-1">รายงานการบำรุงรักษาและทวนสอบ</h2>
                <p class="text-gray-500 mb-6">Temperature Transmitter</p>
                <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
                    <div>
                        <label for="plant" class="block text-sm font-medium text-gray-600 mb-2">โรงงาน</label>
                        <input type="text" id="plant" class="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 transition" placeholder="เช่น อธิมาตร">
                    </div>
                    <div>
                        <label for="department" class="block text-sm font-medium text-gray-600 mb-2">ส่วนงาน</label>
                        <input type="text" id="department" class="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 transition" placeholder="เช่น กลั่นสุรา">
                    </div>
                    <div>
                        <label for="check-date" class="block text-sm font-medium text-gray-600 mb-2">ว/ด/ป ตรวจเช็ค</label>
                        <input type="date" id="check-date" class="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 transition">
                    </div>
                </div>
            </div>

            <div id="equipment-list" class="space-y-6"></div>

            <div class="mt-10">
                 <button id="submit-button" onclick="submitReport()" class="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-6 rounded-lg shadow-md hover:shadow-lg transition-all flex items-center justify-center text-lg">
                    ยืนยันและส่งรายงาน
                </button>
            </div>
        </div>

        <!-- ===== History View ===== -->
        <div id="history-view" class="view hidden-view">
             <div class="bg-white p-6 sm:p-8 rounded-xl shadow-sm border border-gray-200">
                <h2 class="text-xl font-bold text-gray-800 mb-6">ประวัติการส่งรายงาน</h2>
                <div id="history-table-container" class="overflow-x-auto">
                    <!-- History table will be rendered here -->
                </div>
            </div>
        </div>
    </main>

    <!-- Firebase SDK and App Logic -->
    <script type="module">
        // Firebase Imports
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, addDoc, onSnapshot, doc, deleteDoc, serverTimestamp, query, orderBy } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // --- CONFIG and INITIALIZATION ---
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'pm-report-app-standalone';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : { apiKey: "test", authDomain: "test.firebaseapp.com", projectId: "test" };
        
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);

        let currentUser = null;

        // --- MASTER DATA ---
        const equipmentMasterList = [ { code: "TT-012", description: "วัดอุณหภูมิน้ำหล่อเย็นออกจาก Condenser Re2" }, { code: "TT-013", description: "วัดอุณหภูมิน้ำหล่อเย็นออกจาก Condenser R2" }, { code: "TT-015", description: "ท่อส่าส่งเข้าหอกลั่น (วัดอุณหภูมิน้ำส่าเข้าหอกลั่น)" }, { code: "TT-016", description: "ก้นหอ D (วัดอุณหภูมิน้ำกากส่าออกจากก้นหอ D)" }, { code: "TI-019", description: "Condenser (วัดอุณหภูมิน้ำหล่อเย็นเข้า Condenser)" }, { code: "TT-110", description: "วัดอุณหภูมิกั้นหอ D" }, { code: "TI-111", description: "วัดอุณหภูมิ Tray 3 ก้นหอ D" }, { code: "TI-113", description: "วัดอุณหภูมิยอดหอ D" }, { code: "TI-210", "description": "วัดอุณหภูมิกั้นหอ H" }, { code: "TI-211", description: "วัดอุณหภูมิยอดหอ H" }, { code: "TI-310", description: "วัดอุณหภูมิกั้นหอ Re" }, { code: "TI-313", description: "วัดอุณหภูมิ Condenser Re4" }, { code: "TT-410", description: "วัดอุณหภูมิกั้นพอ R" }, { code: "TI-413", description: "วัดอุณหภูมิ Condenser R1" }, { code: "TI-416", description: "วัดอุณหภูมิยอดหอ R" }, ];
        const inspectionItems = ["สายสัญญาณสำหรับ RTD", "ฝาปิดครอบสำหรับ RTD", "บอดี้สำหรับ RTD", "ภายในบอดี้สำหรับ RTD"];

        // --- UI Functions ---
        function showLoading(show, text = "กำลังโหลด...") {
            document.getElementById('loading-text').textContent = text;
            document.getElementById('loading-spinner').style.display = show ? 'flex' : 'none';
        }

        window.switchView = (viewId) => {
            document.querySelectorAll('.view').forEach(v => v.classList.add('hidden-view'));
            document.getElementById(viewId).classList.remove('hidden-view');

            document.querySelectorAll('header nav button').forEach(b => {
                b.classList.remove('text-blue-600', 'border-blue-500');
                b.classList.add('text-gray-500', 'border-transparent', 'hover:border-gray-300', 'hover:text-gray-700');
            });
            const activeTab = document.getElementById(`tab-${viewId.split('-')[0]}`);
            activeTab.classList.add('text-blue-600', 'border-blue-500');
            activeTab.classList.remove('text-gray-500', 'border-transparent', 'hover:border-gray-300', 'hover:text-gray-700');
        };

        function createEquipmentCard(equipment, index) {
            const list = document.getElementById('equipment-list');
            const card = document.createElement('div');
            card.className = "bg-white rounded-xl shadow-sm border border-gray-200 p-6 sm:p-8";
            
            const inspectionButtonsHtml = inspectionItems.map((item, i) =>
                `<div class="flex items-center justify-between py-3 border-b border-gray-200 last:border-b-0">
                    <span class="text-gray-700">${i + 1}. ${item}</span>
                    <div class="flex gap-3" data-status-group="${i}">
                        <button onclick="selectStatus(this, 'pass')" class="status-button pass-button font-semibold py-2 px-6 rounded-lg transition">ผ่าน</button>
                        <button onclick="selectStatus(this, 'fail')" class="status-button fail-button font-semibold py-2 px-6 rounded-lg transition">ไม่ผ่าน</button>
                    </div>
                </div>`).join('');
                
            card.innerHTML = `
                <div class="grid grid-cols-1 md:grid-cols-2 gap-x-6 gap-y-4 mb-6">
                     <div>
                        <label class="block text-sm font-medium text-gray-500">รหัส (Code)</label>
                        <p class="code-text w-full p-3 bg-gray-100 rounded-lg text-gray-800 font-medium mt-1">${equipment.code}</p>
                    </div>
                    <div>
                        <label class="block text-sm font-medium text-gray-500">หลักการทำงาน / ตำแหน่ง</label>
                        <p class="desc-text w-full p-3 bg-gray-100 rounded-lg text-gray-800 font-medium mt-1">${equipment.description}</p>
                    </div>
                </div>

                <h4 class="text-md font-bold text-gray-600 mt-6 mb-2">สถานะรายละเอียดการตรวจสอบ</h4>
                <div class="border border-gray-200 rounded-lg">
                    ${inspectionButtonsHtml}
                </div>

                <div class="mt-6">
                    <label class="block text-sm font-medium text-gray-600 mb-2">รายการแก้ไข / ปรับแต่ง / ซ่อมแซม</label>
                    <textarea rows="3" class="notes-text w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 transition" placeholder="กรอกรายละเอียด (ถ้ามี)..."></textarea>
                </div>`;
            list.appendChild(card);
        }

        window.selectStatus = (button, status) => {
            const group = button.parentElement;
            const passBtn = group.querySelector('.pass-button');
            const failBtn = group.querySelector('.fail-button');

            passBtn.classList.remove('status-button-active-pass');
            failBtn.classList.remove('status-button-active-fail');
            passBtn.classList.add('status-button');
            failBtn.classList.add('status-button');
            
            if (status === 'pass') {
                button.classList.add('status-button-active-pass');
            } else {
                button.classList.add('status-button-active-fail');
            }
        };

        // --- Firebase Functions ---
        const reportsCollection = collection(db, `artifacts/${appId}/public/data/reports`);

        window.submitReport = async () => {
            if (!currentUser) { alert("กรุณารอสักครู่ ระบบกำลังยืนยันตัวตน..."); return; }
            
            showLoading(true, "กำลังส่งรายงาน...");
            const submitButton = document.getElementById('submit-button');
            submitButton.disabled = true;

            const equipmentData = Array.from(document.querySelectorAll('#equipment-list > div')).map(card => {
                const statuses = Array.from(card.querySelectorAll('[data-status-group]')).map(group => {
                    if (group.querySelector('.status-button-active-pass')) return 'pass';
                    if (group.querySelector('.status-button-active-fail')) return 'fail';
                    return 'pending';
                });
                return {
                    code: card.querySelector('.code-text').textContent,
                    description: card.querySelector('.desc-text').textContent,
                    notes: card.querySelector('.notes-text').value,
                    statuses: statuses
                };
            });

            const report = {
                plant: document.getElementById('plant').value || 'ไม่ได้ระบุ',
                department: document.getElementById('department').value || 'ไม่ได้ระบุ',
                checkDate: document.getElementById('check-date').value,
                equipmentData: equipmentData,
                submitterId: currentUser.uid,
                createdAt: serverTimestamp()
            };
            
            try {
                await addDoc(reportsCollection, report);
                alert('ส่งรายงานเรียบร้อยแล้ว!');
                resetForm();
                switchView('history-view');
            } catch (error) {
                alert("เกิดข้อผิดพลาดในการบันทึก: " + error.message);
            } finally {
                showLoading(false);
                submitButton.disabled = false;
            }
        };
        
        function resetForm() {
            document.getElementById('equipment-list').innerHTML = '';
            equipmentMasterList.forEach(createEquipmentCard);
            document.getElementById('plant').value = '';
            document.getElementById('department').value = '';
            document.getElementById('check-date').valueAsDate = new Date();
        }
        
        function renderHistory(reports) {
            const container = document.getElementById('history-table-container');
            if (!reports || reports.length === 0) {
                container.innerHTML = '<div class="text-center py-16"><svg class="mx-auto h-12 w-12 text-gray-400" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true"><path vector-effect="non-scaling-stroke" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 13h6m-3-3v6m-9 1V7a2 2 0 012-2h6l2 2h6a2 2 0 012 2v8a2 2 0 01-2 2H5a2 2 0 01-2-2z" /></svg><h3 class="mt-2 text-sm font-medium text-gray-900">ยังไม่มีรายงาน</h3><p class="mt-1 text-sm text-gray-500">เริ่มกรอกรายงานใหม่ในแท็บ "กรอกรายงาน"</p></div>';
                return;
            }
            let tableHtml = `<table class="min-w-full divide-y divide-gray-200">
                <thead class="bg-gray-50"><tr>
                    <th scope="col" class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">วันที่ตรวจ</th>
                    <th scope="col" class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">โรงงาน</th>
                    <th scope="col" class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">ผู้ส่ง</th>
                    <th scope="col" class="relative px-6 py-3"><span class="sr-only">จัดการ</span></th>
                </tr></thead><tbody class="bg-white divide-y divide-gray-200">`;
            
            reports.forEach(report => {
                tableHtml += `<tr class="hover:bg-gray-50 transition">
                    <td class="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900">${new Date(report.checkDate).toLocaleDateString('th-TH') || 'N/A'}</td>
                    <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-600">${report.plant}</td>
                    <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-600 truncate" title="${report.submitterId}">${report.submitterId}</td>
                    <td class="px-6 py-4 text-center">
                      <button onclick="deleteReport('${report.id}')" class="text-red-500 hover:text-red-700 font-medium transition-colors" title="ลบรายงาน">
                        <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" /></svg>
                      </button>
                    </td>
                </tr>`;
            });
            tableHtml += `</tbody></table>`;
            container.innerHTML = tableHtml;
        }
        
        window.deleteReport = async (docId) => {
            if (confirm('คุณแน่ใจหรือไม่ว่าต้องการลบรายงานนี้?')) {
                showLoading(true, "กำลังลบรายงาน...");
                try {
                    await deleteDoc(doc(db, `artifacts/${appId}/public/data/reports`, docId));
                    alert('ลบรายงานเรียบร้อยแล้ว');
                } catch (error) {
                    alert("เกิดข้อผิดพลาดในการลบ: " + error.message);
                } finally {
                    showLoading(false);
                }
            }
        };

        // --- App Initialization ---
        document.addEventListener('DOMContentLoaded', async () => {
            document.getElementById('check-date').valueAsDate = new Date();
            equipmentMasterList.forEach(createEquipmentCard);

            onAuthStateChanged(auth, async (user) => {
                if (user) {
                    currentUser = user;
                    const q = query(reportsCollection, orderBy("createdAt", "desc"));
                    onSnapshot(q, (snapshot) => {
                        const reports = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                        renderHistory(reports);
                    });
                    showLoading(false);
                }
            });

            try {
                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }
            } catch (error) {
                console.error("Authentication failed:", error);
                document.getElementById('loading-spinner').innerHTML = '<p class="text-red-500">Authentication Failed. Please refresh.</p>';
            }
        });
    </script>
</body>
</html>
