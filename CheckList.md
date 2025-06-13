## Check list 

## Nguyên tắc chung

Mỗi record có ít nhất các field:

  - id (UUID)

  - updated_at (timestamp hoặc version counter)

  - synced_at (local: thời điểm record sync gần nhất)

  - is_deleted (để soft-delete)

  - sync_status (synced, created, updated, deleted, conflict)


So sánh updated_at hoặc version để xác định thay đổi gần nhất.

**1. Người dùng tạo mới data khi offline**

Local : Lưu data với sync_status = create

Khi online : 

- Gửi request để update lên server
- Thành công sẽ cập nhật id server và cập nhật dưới local sync_statsus = synced 
- Khi sync thất bại thì sẽ có cơ chế retry

**2. Người dùng cập nhật data khi offline**

Local: record sync_status = updated, updated_at = Date.now()

Khi online:

- Kiểm tra updated_at so với bản trên server:
- Nếu local mới hơn → Gửi PUT/PATCH lên server và cập nhật dưới local sync_statsus = synced 
- Nếu server mới hơn → Conflict
  - Cập nhật sync_status = conflict.
  - (Optional) Hiển thị UI yêu cầu người dùng resolve (chọn local/server hoặc merge).
- Khi sync thất bại thì sẽ có cơ chế retry

**3. Người dùng xoá data khi offline**

Local: is_deleted = true, sync_status = deleted

Khi online:

- Gửi DELETE (soft hoặc hard) lên server.
- Nếu server phản hồi thành công thì xoá hẳn ở dưới local hoặc giữ is_deleted = true.
- Khi sync thất bại thì sẽ có cơ chế retry

#### Code mẫu cho case 1, 2 và 3


```
async function syncLocalToRemote(db) {
  const localChanges = await getLocalChanges(db);

  for (const item of localChanges) {
    try {
      if (item.sync_status === SYNC_STATUS.CREATED) {
        const res = await api.post('/items', item); // POST to server
        await markAsSynced(db, item.id, res.updated_at);
      } else if (item.sync_status === SYNC_STATUS.UPDATED) {
        const remote = await api.get(`/items/${item.id}`);
        if (item.updated_at > remote.updated_at) {
          await api.put(`/items/${item.id}`, item); // update server
          await markAsSynced(db, item.id, item.updated_at);
        } else {
          // conflict!
          await markAsConflict(db, item.id);
        }
      } else if (item.sync_status === SYNC_STATUS.DELETED) {
        await api.delete(`/items/${item.id}`);
        await deleteLocalItem(db, item.id);
      }
    } catch (e) {
      console.warn('Sync failed:', e);
    }
  }
}

function getLocalChanges(db) {
  return new Promise((resolve, reject) => {
    db.transaction(tx => {
      tx.executeSql(
        `SELECT * FROM items WHERE sync_status != ?`,
        [SYNC_STATUS.SYNCED],
        (_, { rows }) => resolve(rows._array),
        (_, err) => reject(err),
      );
    });
  });
}

```

#### Lắng nghe khi online

```
useEffect(() => {
  const unsubscribe = NetInfo.addEventListener(state => {
    if (state.isConnected) {
      syncLocalToRemote(db);
    }
  });

  return () => unsubscribe();
}, []);

```

#### Sync định kỳ

```
useEffect(() => {
  const interval = setInterval(() => {
    syncAll(db);
  }, 60 * 1000); // mỗi 60 giây

  return () => clearInterval(interval); // clear khi unmount
}, []);

```

**4. Người dùng tạo mới data khi online**

- Gọi API thêm data
- Nếu thành công thì sẽ insert vào local với sync_status = synced

**5. Người dùng cập nhật data khi online**

- Gọi API cập nhật data
- Nếu thành công thì sẽ update local với sync_status = synced
- local update_at = server update_at

**6. Người dùng xoá data khi online**

- Gọi API xoá data
- Nếu thành công thì sẽ xoá ở dưới local hoặc chuyển is_deleted = true

**7. Data dạng dánh sách có sử dụng pagination**

Đồng bộ data

| Hành động            | Nơi lưu | sync\_status          | sync hướng nào | Chi tiết                                                    |
| -------------------- | ------- | --------------------- | -------------- | ----------------------------------------------------------- |
| Tạo mới offline      | SQLite  | `'created'`           | local → remote | Gửi POST khi online, nếu OK → `synced`                      |
| Sửa offline          | SQLite  | `'updated'`           | local → remote | Gửi PUT nếu `updated_at` local mới hơn                      |
| Xoá offline          | SQLite  | `'deleted'`           | local → remote | Gửi DELETE nếu đã sync trước đó                             |
| Server có bản mới    | Server  | (server `updated_at`) | remote → local | Nếu `updated_at` server > local và local không sửa → update |
| Server có bản bị xoá | Server  | (flag `is_deleted`)   | remote → local | Gỡ bỏ hoặc flag bản ghi local nếu chưa sửa                  |


| Trạng thái     | Dữ liệu hiển thị       | Nguồn      | Ghi chú                              |
| -------------- | ---------------------- | ---------- | ------------------------------------ |
| Online lần đầu | Server → lưu vào local | Remote     | Ghi nhận `updated_at`, `sync_status` |
| Offline        | SQLite query + OFFSET  | Local only | `ORDER BY updated_at DESC`           |
| Online lại     | Push local changes     | Two-way    | So sánh `updated_at` hoặc version    |


## Edge case

**1. Tạo mới rồi xoá khi offline**
- Khi chưa được sync thì sẽ xoá hoàn toàn ra khỏi local

**2. Conflict khi thao tác thêm/xoá/sửa trên một tài khoản đang online trên nhiều thiết bị khác cùng một thời điểm**
  - Khi user sửa thì sẽ nhận lỗi -> Đánh dấu dữ liệu đó dưới local thành is_deleted = true hoặc đánh dấu là conflict
  - Hoặc sử dụng realtime để đẩy các sự kiện tới tất cả thiết bị đang online và thực hiện lệnh đồng bộ
  - Hoặc thực hiện sync định kỳ

**3. Dữ liệu dưới local quá cũ chưa được đồng bộ**

Trong thời gian offline, dữ liệu có thể đã thay đổi trên server:
  - Có item bị xoá
  - Có item bị sửa
  - Có item mới chèn vào đầu danh sách

Khi online trở lại nếu dữ liệu đồng bộ khác quá nhiều thì nên có thông báo cần cập nhật danh sách

**4. Đồng bộ gây trùng ID**
- Nên sử dụng id tự sinh, được server trả về để cập nhât lại id trong local

**5. Server không hỗ trợ soft-delete**
- Đề xuất đồng bộ logic giữa FE và BE

**6. Mạng yếu hoặc dữ liệu cần đồng bộ quá lớn**
- Chia nhỏ từng item hoặc theo từng batch nhỏ
- Đặt delay mỗi lần sync nếu cần

**7. Thay đổi thứ tự hiển thị (sort) gây lỗi pagination**
- Luôn sync theo update_at desc
- Khi online trở lại mà dữ liệu có thay đổi thì thông báo cần cập nhật dữ liệu

**8. Dữ liệu được sửa trên cả 2 thiết bị cùng lúc**
- Có thể tự merge nếu cả 2 update khác trường dữ liệu
- Nên có tính năng báo conflict và lưu lại lịch sử chỉnh sửa để dễ dàng phục hồi

