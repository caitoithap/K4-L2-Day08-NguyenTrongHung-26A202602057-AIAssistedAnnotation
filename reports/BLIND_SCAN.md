# Quét độc lập trước khi xem pre-label

Frame:`\to_label\round1\images\train\frame_0392.jpg`

Số xe nhìn thấy bằng mắt: 26

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe:

1. Mép trái phía giữa ảnh, ngay sát dải phân cách/tường tối: một xe con màu tối đang chạy về phía camera, bị che khuất một phần nên rất dễ bị bỏ sót hoặc vẽ thiếu bbox.
2. Góc dưới giữa ảnh, sát mép dưới khung hình: một xe con màu tối chỉ lộ phần nóc/thân trên, bị cắt bởi mép ảnh nên dễ bị AI bỏ sót hoặc vẽ bbox không đầy đủ.

Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.
