# World Craft — Tổng quan hoạt động

World Craft là framework đa-agent biến mô tả văn bản (vd: *"một lâu đài cổ u ám có vài nhân vật bí ẩn"*) thành một **scene 2.5D chạy được trong Godot**, kèm NPC có "linh hồn" (LLM-driven personality + lịch sinh hoạt). Hệ thống gồm **2 phần lớn**:

- **World Guild** (Python) – "bộ não": chuỗi agent sinh kế hoạch scene (JSON), vẽ asset, viết soul.
- **World Scaffold** (Godot/GDScript) – "bộ xương": TCP server nhận JSON rồi dựng scene runtime.

## Diagram tổng quan (ASCII)

```
                              ┌──────────────────────────┐
   user prompt (text) ───────▶│   Enricher Agent  (LLM)  │  prompt → blueprint không gian
                              └─────────────┬────────────┘     (zones, đồ đạc, 2.5D)
                                            │ enriched prompt
                                            ▼
              ┌───────────────────────────────────────────────────────────┐
              │             generate_and_iterate_scene  (HIGH LOOP)       │
              │                                                           │
              │   ┌─────────────────────────────────────────────────┐     │
              │   │   _run_manager_with_validation (INNER LOOP x3)  │     │
              │   │                                                 │     │
              │   │   ┌──────────────────┐                          │     │
              │   │   │ Manager Agent    │  get_scene_plan /        │     │
              │   │   │     (LLM)        │  repair_scene_plan       │     │
              │   │   └────────┬─────────┘                          │     │
              │   │            │ scene plan JSON                    │     │
              │   │            ▼                                    │     │
              │   │   ┌──────────────────┐                          │     │
              │   │   │ Hard-Rule Fixer  │  ép base/visual_size     │     │
              │   │   │  (code, no LLM)  │  cho floor v.v.          │     │
              │   │   └────────┬─────────┘                          │     │
              │   │            ▼                                    │     │
              │   │   ┌──────────────────┐    pass? ──► return plan │     │
              │   │   │ Validator (code) │                          │     │
              │   │   │  • asset_id ref  │    fail ──► error report │     │
              │   │   │  • AABB collide  │            (feed Manager)│     │
              │   │   └────────┬─────────┘                          │     │
              │   │            ▼                                    │     │
              │   └─────────────────────────────────────────────────┘     │
              │                          │ valid plan                     │
              │                          ▼                                │
              │            ┌──────────────────────────┐                   │
              │            │   Critic Agent  (VLM)    │ vẽ sketch top-    │
              │            │  • size sanity           │ down (PIL) +      │
              │            │  • semantic placement    │ bảng size →       │
              │            │  • render top-down PNG   │ multimodal LLM    │
              │            └────────────┬─────────────┘                   │
              │                         │  ok? ──► thoát                  │
              │                         │  fix? ──► quay lại INNER LOOP   │
              └─────────────────────────┴────────────────────────────────-┘
                                            │ final scene plan
                                            ▼
                       ┌─────────────────────────────────────────┐
                       │   Artist Agent  (parallel, 5 workers)   │
                       │   ┌──────────┐ ┌──────────┐ ┌────────┐  │
                       │   │ walls/   │ │ objects  │ │ NPC/   │  │
                       │   │ floors:  │ │ : LLM    │ │ agent: │  │
                       │   │ OpenCV   │ │ img-gen  │ │ edit   │  │
                       │   │ procedural│ │ + retr. │ │ base   │  │
                       │   └──────────┘ └────┬─────┘ │ sprite │  │
                       │                     │       └────────┘  │
                       │      Asset Retriever│                    │
                       │      ← asset_index.json (build_asset_index)
                       │                     ▼                    │
                       │    <godot>/generated_assets/*.png         │
                       └─────────────────────┬───────────────────┘
                                             ▼
                  ┌─────────────────────────────────────────────┐
                  │       Soul Writer Agent  (LLM)              │
                  │  • npc_souls/<name>.json (personality,      │
                  │    goals, dialogue, schedule, API config)   │
                  │  • world/world_context.json                 │
                  └─────────────────────────┬───────────────────┘
                                            ▼
                          save_scene_to_file → saved_levels/*.json
                                            │
                                            ▼  TCP 127.0.0.1:8080
                                  ┌──────────────────────┐
                                  │  godot_client        │  JSON command
                                  │  send_command()      │  {action:
                                  └──────────┬───────────┘   build_scene_from_json}
                                             ▼
              ╔════════════════════════════════════════════════════╗
              ║   WORLD SCAFFOLD  —  Godot scene_builder_server.gd ║
              ║                                                    ║
              ║   build_scene_procedurally(plan):                  ║
              ║     • dựng TileSet từ generated_assets/*.png        ║
              ║     • vẽ floor_layer + wall sprites (offset/scale) ║
              ║     • spawn objects (Sprite2D + collision/nav)     ║
              ║     • spawn npc.tscn / agent.tscn với soul_file    ║
              ║     • bake NavigationRegion2D + start WorldClock   ║
              ╚════════════════════════════════════════════════════╝
                                             │
                                             ▼
                            🎮  AI Town chạy được, NPC tự sống
```

## Giải thích luồng

**1. Enricher** mở rộng câu prompt thô thành "bản thiết kế không gian" (zone, vị trí tương đối, ràng buộc 2.5D) — đầu vào tốt hơn cho Manager.

**2. Vòng lặp 2 tầng sinh `scene_plan` (JSON)**:
- *Inner loop (≤3 lần)*: **Manager** (LLM) sinh hoặc sửa plan → **Hard-Rule Fixer** ép các giá trị cứng (vd floor phải `base_size [1,1], visual_size [2,2]`) → **Validator** (code thuần) kiểm tra `asset_id` có tồn tại và **AABB collision** giữa object/NPC. Lỗi → feed lại Manager.
- *Outer loop*: khi plan đã sạch về vật lý, **Critic** (VLM) render sketch top-down bằng PIL, gửi kèm bảng kích thước cho LLM thị giác kiểm tra *ngữ nghĩa* (toilet trong bếp? cửa bị chặn? phòng trống?). Nếu có gợi ý → quay lại inner loop để sửa. Đây chính là **"spatial error correction via reverse engineering"** mà paper nhấn mạnh.

**3. Artist Agent** chạy song song 5 worker, định tuyến theo loại asset:
- wall/floor → texture procedural OpenCV (tách `_top`/`_side` cho tường),
- object → LLM image-gen, có **Asset Retriever** tìm ảnh tham chiếu gần nhất từ `asset_index.json` (token-set + Manhattan distance trên visual_size),
- NPC/agent → edit base sprite sheet (`male_base.png` ...).

**4. Soul Writer** sinh hồ sơ "linh hồn" cho mỗi NPC (tính cách, mục tiêu, lịch sinh hoạt từ `semantic_tag`) và `world_context.json` để các agent "nhìn thấy" thế giới. Agent (vs NPC thường) còn được nhúng cấu hình API riêng để tự gọi LLM lúc runtime.

**5. Save + gửi Godot**: lưu JSON cuối vào `saved_levels/`, rồi `godot_client` mở TCP socket tới `127.0.0.1:8080`. Phía Godot, `scene_builder_server.gd` build TileSet động, đặt sprite (bottom-center anchor, logic gắn tường), bake navigation và khởi động `WorldClock` — scene sẵn sàng chơi.

**Điểm thiết kế đáng chú ý**: tách bạch **kiểm tra code-level (Validator, deterministic, nhanh)** với **kiểm tra semantic (Critic, VLM, đắt)**, và chèn một bước **hard-rule fixer** không-LLM giữa các vòng để LLM không lặp lại sai số quy chuẩn.
