# 📍 Hệ Thống Điểm Danh GPS (GitHub Pages + Google Sheet)

Hệ thống điểm danh miễn phí bằng GPS, chạy trên **GitHub Pages** và lưu dữ liệu vào **Google Sheet** thông qua **Google Apps Script Web App**.

✅ Xác thực vị trí GPS  
✅ Lưu lịch sử vào Google Sheet  
✅ Xuất dữ liệu từ Google Sheet ra Excel (.xlsx)  
✅ Chạy hoàn toàn miễn phí  

---

## 🚀 Demo hoạt động

- Người dùng mở trang web
- Cho phép GPS
- Nhập họ tên + mã học sinh (tùy chọn)
- Bấm **Điểm Danh**
- Dữ liệu sẽ lưu vào Google Sheet
- Bấm nút **Sheet** để tải Excel từ Google Sheet

---

## 📌 Công nghệ sử dụng

- HTML / CSS / JavaScript (Frontend)
- Google Apps Script (Backend API)
- Google Sheets (Database)
- GitHub Pages (Hosting)

---

# 1️⃣ Cài đặt Google Apps Script (Backend)

## Bước 1: Tạo Google Sheet
1. Tạo 1 Google Sheet mới
2. Đặt tên file tùy ý (VD: `DiemDanhData`)

---

## Bước 2: Tạo Apps Script
1. Vào Google Sheet
2. Chọn **Extensions → Apps Script**
3. Xóa code mặc định
4. Paste toàn bộ code Apps Script vào
## 📌 Code Google Apps Script

```js
// ==============CODE APPS SCRIPT =============================
// CẤU HÌNH — Chỉnh 4 dòng này rồi deploy 1 lần duy nhất
// ============================================================
const CONFIG = {
  LAT: 10.0452,          // ← Vĩ độ trung tâm (công ty/trường)
  LNG: 105.7469,         // ← Kinh độ trung tâm
  RADIUS_M: 200,         // ← Bán kính cho phép (mét)
  SHEET_NAME: "DiemDanh" // ← Tên sheet ghi dữ liệu
};

// ============================================================
// doPost — Nhận điểm danh từ frontend
// ============================================================
function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);
    const { name, employeeId, lat, lng } = data;

    if (!name || lat === undefined || lng === undefined) {
      return jsonOut({ success: false, message: "Thiếu dữ liệu!" });
    }

    const dist    = haversine(CONFIG.LAT, CONFIG.LNG, parseFloat(lat), parseFloat(lng));
    const allowed = dist <= CONFIG.RADIUS_M;
    const now     = new Date();

    logToSheet(name, employeeId || "—", lat, lng, dist, allowed, now);

    return jsonOut({
      success:   allowed,
      distance:  Math.round(dist),
      maxRadius: CONFIG.RADIUS_M,
      timestamp: Utilities.formatDate(now, "Asia/Ho_Chi_Minh", "dd/MM/yyyy HH:mm:ss"),
      message:   allowed
        ? `✅ Điểm danh thành công! Cách ${Math.round(dist)}m`
        : `❌ Ngoài vùng cho phép! Cách ${Math.round(dist)}m (tối đa ${CONFIG.RADIUS_M}m)`
    });

  } catch (err) {
    return jsonOut({ success: false, message: "Lỗi server: " + err.message });
  }
}

// ============================================================
// doGet — Ping kiểm tra hoạt động + Xuất dữ liệu cho frontend
// ============================================================
function doGet(e) {
  const action = e && e.parameter && e.parameter.action;

  // ?action=export → trả về toàn bộ dữ liệu sheet dạng JSON
  if (action === "export") {
    try {
      const ss    = SpreadsheetApp.getActiveSpreadsheet();
      const sheet = ss.getSheetByName(CONFIG.SHEET_NAME);

      if (!sheet) {
        return jsonOut({ success: false, message: "Sheet chưa có dữ liệu." });
      }

      const values  = sheet.getDataRange().getValues();
      if (values.length <= 1) {
        return jsonOut({ success: true, rows: [] });
      }

      const headers = values[0];
      const rows    = values.slice(1).map(row => {
        const obj = {};
        headers.forEach((h, i) => {
          obj[h] = row[i] instanceof Date
            ? Utilities.formatDate(row[i], "Asia/Ho_Chi_Minh", "dd/MM/yyyy HH:mm:ss")
            : row[i];
        });
        return obj;
      });

      return jsonOut({ success: true, rows });

    } catch (err) {
      return jsonOut({ success: false, message: "Lỗi xuất dữ liệu: " + err.message });
    }
  }

  return jsonOut({ success: true, status: "API đang hoạt động ✓", sheetName: CONFIG.SHEET_NAME });
}

// ============================================================
// HELPER — Haversine (khoảng cách 2 tọa độ, đơn vị mét)
// ============================================================
function haversine(lat1, lon1, lat2, lon2) {
  const R    = 6371000;
  const toRad = x => x * Math.PI / 180;
  const dLat  = toRad(lat2 - lat1);
  const dLon  = toRad(lon2 - lon1);
  const a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) *
    Math.sin(dLon / 2) * Math.sin(dLon / 2);
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
}

// ============================================================
// HELPER — Ghi log vào Google Sheet
// ============================================================
function logToSheet(name, employeeId, lat, lng, dist, allowed, time) {
  const ss    = SpreadsheetApp.getActiveSpreadsheet();
  let   sheet = ss.getSheetByName(CONFIG.SHEET_NAME);

  if (!sheet) {
    sheet = ss.insertSheet(CONFIG.SHEET_NAME);
    const header = [["Thời gian", "Họ tên", "Mã NV", "Latitude", "Longitude", "Khoảng cách (m)", "Trạng thái"]];
    sheet.getRange(1, 1, 1, 7).setValues(header);
    sheet.getRange(1, 1, 1, 7)
      .setFontWeight("bold")
      .setBackground("#1a73e8")
      .setFontColor("#ffffff");
    sheet.setFrozenRows(1);
  }

  const status = allowed ? "✓ Có mặt" : "✗ Vắng mặt";
  sheet.appendRow([
    Utilities.formatDate(time, "Asia/Ho_Chi_Minh", "dd/MM/yyyy HH:mm:ss"),
    name,
    employeeId,
    parseFloat(lat).toFixed(6),
    parseFloat(lng).toFixed(6),
    Math.round(dist),
    status
  ]);

  if (!allowed) {
    const lastRow = sheet.getLastRow();
    sheet.getRange(lastRow, 1, 1, 7).setBackground("#fce8e6");
  }
}

// ============================================================
// HELPER — Trả về JSON
// ============================================================
function jsonOut(data) {
  return ContentService
    .createTextOutput(JSON.stringify(data))
    .setMimeType(ContentService.MimeType.JSON);
}
```



---

## Bước 3: Cấu hình tọa độ và bán kính
Trong Apps Script, sửa phần:

```js
const CONFIG = {
  LAT: 10.0452,
  LNG: 105.7469,
  RADIUS_M: 200,
  SHEET_NAME: "DiemDanh"
};
```

📌 Ý nghĩa:
- `LAT` và `LNG`: tọa độ trung tâm (trường / công ty)
- `RADIUS_M`: bán kính cho phép điểm danh (mét)
- `SHEET_NAME`: tên sheet lưu dữ liệu

---

## Bước 4: Deploy Web App
1. Bấm **Deploy → New deployment**
2. Chọn loại: **Web app**
3. Cấu hình:
   - **Execute as:** `Me`
   - **Who has access:** `Anyone`
4. Bấm **Deploy**
5. Copy URL dạng:

```
https://script.google.com/macros/s/xxxxxxx/exec
```

👉 Đây là API URL dùng để paste vào frontend.

---

# 2️⃣ Cài đặt GitHub Pages (Frontend)

## Bước 1: Tạo Repository GitHub
1. Vào GitHub → New repository
2. Tạo repo ví dụ: `diemdanh-gps`
3. Upload file `index.html` vào repo

---

## Bước 2: Bật GitHub Pages
1. Vào repo GitHub → **Settings**
2. Chọn **Pages**
3. Source:
   - Branch: `main`
   - Folder: `/root`
4. Save

Sau đó GitHub sẽ cấp link dạng:

```
https://yourname.github.io/diemdanh-gps/
```

---

# 3️⃣ Hướng dẫn sử dụng hệ thống

## Bước 1: Mở website GitHub Pages
Người dùng mở link website trên điện thoại hoặc máy tính.

---

## Bước 2: Cho phép quyền GPS
Trình duyệt sẽ yêu cầu quyền vị trí → chọn **Allow / Cho phép**.

---

## Bước 3: Cấu hình API URL (chỉ cần làm 1 lần)
Lần đầu mở trang web sẽ hiện phần:

⚙️ Cần cấu hình API URL

➡️ Paste URL Apps Script Web App đã deploy:

```
https://script.google.com/macros/s/xxxx/exec
```

Bấm **Lưu & tiếp tục**

Hệ thống sẽ lưu URL này vào trình duyệt.

---

## Bước 4: Điểm danh
1. Nhập **Họ và tên**
2. Nhập **Mã học sinh** (tùy chọn)
3. Bấm **Điểm Danh**

Nếu trong vùng cho phép → thành công  
Nếu ngoài vùng → bị từ chối

---

# 4️⃣ Xuất dữ liệu Excel từ Google Sheet

Bấm nút **Sheet** trên giao diện:

📌 Hệ thống sẽ:
- tải dữ liệu từ Google Sheet
- xuất thành file Excel `.xlsx`
- tự động download về máy

Tên file ví dụ:

```
DiemDanh_GoogleSheet_2026-04-23.xlsx
```

---

# 5️⃣ Cấu trúc dữ liệu trong Google Sheet

Sheet sẽ tự tạo header:

| Thời gian | Họ tên | Mã NV | Latitude | Longitude | Khoảng cách (m) | Trạng thái |
|----------|--------|-------|----------|-----------|-----------------|-----------|

---

# 6️⃣ Lưu ý quan trọng

## ⚠️ Nếu bấm điểm danh nhưng không ghi được vào Sheet
- Kiểm tra Apps Script đã deploy Web App chưa
- Kiểm tra quyền deploy phải là **Anyone**
- Kiểm tra API URL đúng dạng `/exec`

---

## ⚠️ Nếu GPS sai hoặc không lấy được
- Bật GPS trên điện thoại
- Cho phép trình duyệt quyền Location
- Nếu dùng iPhone: bật Location trong Safari Settings

---

# 7️⃣ Reset API URL (nếu muốn đổi server)
Nếu bạn muốn đổi API URL mới:
- mở DevTools → Application → LocalStorage
- xóa key:

```
diemdanh_api_url
```

hoặc mở trình duyệt ở chế độ ẩn danh và cấu hình lại.

---

# 📌 Tác giả
Thu Thảo (Tannie)
