#!/usr/bin/env python3
"""
prep_diemthi_2026.py
Tầng dữ liệu sạch cho sản phẩm phân tích/visualization phổ điểm THPT 2026.

Pipeline: đọc CSV toàn quốc -> DROP SBD (bỏ định danh ngay) -> ép kiểu điểm
-> tính sẵn các tổ hợp xét tuyển -> lưu parquet để query nhanh.

Chạy:
    python3 -m venv .venv
    .venv/bin/pip install pandas pyarrow
    .venv/bin/python prep_diemthi_2026.py --in data/diem_thi_thptqg_2026_all.csv --out diemthi_2026.parquet
"""

import argparse
import pandas as pd

# Các môn điểm (khớp schema CSV gốc)
SUBJECTS = [
    "toan", "ngu_van", "ngoai_ngu", "vat_li", "hoa_hoc", "sinh_hoc",
    "lich_su", "dia_li", "gdcd", "tin_hoc", "cong_nghe",
]

# Tổ hợp xét tuyển: tên -> 3 môn cấu thành.
# Chỉ tính tổng khi thí sinh có đủ CẢ 3 môn (nếu thiếu 1 môn -> NaN, không xét).
TO_HOP = {
    "A00": ["toan", "vat_li", "hoa_hoc"],
    "A01": ["toan", "vat_li", "ngoai_ngu"],
    "A02": ["toan", "vat_li", "sinh_hoc"],
    "B00": ["toan", "hoa_hoc", "sinh_hoc"],
    "C00": ["ngu_van", "lich_su", "dia_li"],
    "C01": ["ngu_van", "toan", "vat_li"],
    "D01": ["ngu_van", "toan", "ngoai_ngu"],
    "D07": ["toan", "hoa_hoc", "ngoai_ngu"],
    # Tổ hợp mới liên quan môn Tin học / Công nghệ (format 2025+)
    "A0T": ["toan", "vat_li", "tin_hoc"],
    "K01": ["toan", "ngu_van", "tin_hoc"],
}


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--in", dest="inp", default="data/diem_thi_thptqg_2026_all.csv")
    ap.add_argument("--out", dest="out", default="diemthi_2026.parquet")
    args = ap.parse_args()

    # 1) Đọc. Ép province_code là str để giữ số 0 đứng đầu; sbd đọc rồi bỏ ngay.
    df = pd.read_csv(args.inp, dtype={"sbd": str, "province_code": str})
    print(f"[load] {len(df):,} dòng, {df.shape[1]} cột")

    # 2) ANONYMIZE: bỏ số báo danh — định danh trực tiếp duy nhất.
    if "sbd" in df.columns:
        df = df.drop(columns=["sbd"])
        print("[anon] đã bỏ cột sbd")

    # 3) Ép các cột điểm về số (ô trống '' -> NaN = không thi môn đó).
    for col in SUBJECTS:
        if col in df.columns:
            df[col] = pd.to_numeric(df[col], errors="coerce")

    # 4) Enrich: tính điểm từng tổ hợp (NaN nếu thiếu bất kỳ môn nào).
    for name, cols in TO_HOP.items():
        cols = [c for c in cols if c in df.columns]
        if len(cols) == 3:
            block = df[cols]
            total = block.sum(axis=1)
            total[block.isna().any(axis=1)] = pd.NA  # thiếu môn -> loại
            df[f"th_{name}"] = total

    # 5) Cột phụ tiện phân tích: số môn thí sinh dự thi.
    df["so_mon"] = df[[c for c in SUBJECTS if c in df.columns]].notna().sum(axis=1)

    # 6) Lưu parquet (nén, query nhanh).
    df.to_parquet(args.out, index=False)
    print(f"[save] {args.out}  ({len(df):,} dòng, {df.shape[1]} cột)")

    # Sanity check nhanh
    print("\n[check] TB điểm Toán theo tỉnh (top 5):")
    print(df.groupby("province_name")["toan"].mean().sort_values(ascending=False).head())
    print("\n[check] số thí sinh xét được từng tổ hợp:")
    for name in TO_HOP:
        col = f"th_{name}"
        if col in df.columns:
            print(f"  {name}: {df[col].notna().sum():,}")


if __name__ == "__main__":
    main()
