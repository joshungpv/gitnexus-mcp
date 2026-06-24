# Hướng dẫn Tích hợp và Tối ưu hóa GitNexus, MCP & Hooks cho Claude CLI và Antigravity IDE

Tài liệu này tổng hợp kiến thức từ các tài liệu cấu hình, so sánh các cấp độ tích hợp và đưa ra chiến lược tối ưu nhất khi phát triển đồng thời trên **Claude CLI (Claude Code)** và **Antigravity IDE (Gemini Client)**.

---

## 1. Khái niệm Cốt lõi (Core Concepts)

Để xây dựng một môi trường phát triển thông minh với AI, chúng ta cần hiểu rõ 3 thành phần công nghệ sau:

```mermaid
flowchart LR
    A[GitNexus <br/>Code Intelligence] <-->|Giao thức MCP| B[MCP Server <br/>cung cấp Tools]
    B <--> C{AI Clients}
    C -->|Terminal CLI| D[Claude Code <br/>MCP + Hooks]
    C -->|Visual IDE| E[Antigravity IDE <br/>MCP-only]

    style A fill:#1e1b4b,color:#fff,stroke:#4338ca
    style B fill:#172554,color:#fff,stroke:#1d4ed8
    style C fill:#1e293b,color:#fff,stroke:#475569
    style D fill:#064e3b,color:#fff,stroke:#047857
    style E fill:#701a75,color:#fff,stroke:#a21caf
```

*   **GitNexus:** Hệ thống Code Intelligence phân tích toàn bộ cấu trúc dự án (Monorepo/Multi-project). Nó sinh ra AST (Abstract Syntax Tree), xây dựng call graph, phân tích luồng thực thi và tính toán phạm vi ảnh hưởng (blast radius) khi chỉnh sửa code.
*   **Model Context Protocol (MCP):** Giao thức mở giúp kết nối AI Agent với các ứng dụng bên ngoài. GitNexus đóng vai trò là một MCP Server cung cấp các công cụ: `query` (tìm kiếm ngữ nghĩa), `context` (lấy ngữ cảnh symbol), `impact` (phân tích ảnh hưởng), `detect_changes` (phát hiện thay đổi).
*   **Lifecycle Hooks (Chỉ có ở Claude CLI):** Cơ chế cho phép can thiệp vào vòng đời gọi công cụ của Agent. Nó gồm `PreToolUse` (chạy script trước khi gọi công cụ để bổ sung dữ liệu đầu vào) và `PostToolUse` (chạy script sau khi thực hiện công cụ để kiểm tra hoặc cập nhật trạng thái).

---

## 2. Phân bổ các Cấp độ Cấu hình MCP (Gemini/Antigravity)

Dưới đây là bảng đã được hiệu chỉnh lại các tham số kỹ thuật chính xác theo Tài liệu Plugins & Skills của Google Antigravity để bạn áp dụng trực tiếp:

| Cấp độ cấu hình | Đường dẫn lưu trữ chính xác | Phạm vi tác động | Độ ưu tiên | Khuyên dùng cho |
| :--- | :--- | :--- | :--- | :--- |
| **Cấp Dự án (Project)** | `<project-root>/.mcp.json` | Chỉ áp dụng khi mở dự án hiện tại. | 1 (Cao nhất) | Các công cụ gắn liền với codebase của dự án (ví dụ: DB Reader nội bộ, script tự động hóa). |
| **Cấp Hệ thống hợp nhất** | `~/.gemini/config/mcp_config.json` | Toàn bộ các công cụ thuộc hệ sinh thái Gemini/Antigravity trên máy (IDE + CLI). | 2 (Trung bình) | Tri thức cá nhân toàn cục, công cụ hệ thống bảo mật có chứa API key cá nhân (như Obsidian MCP, Google Search API). |
| **Cấp CLI Độc lập** | `~/.gemini/antigravity-cli/mcp_config.json` | Chỉ áp dụng riêng khi gọi các tiến trình chạy từ Terminal qua Antigravity CLI. | 3 (Thấp nhất) | Các plugin tự động hóa terminal, quản lý container, sandbox phục vụ riêng cho môi trường dòng lệnh. |

> [!IMPORTANT]
> **Quy tắc phân bổ:** Tuyệt đối không khai báo cấu hình MCP của một dự án cụ thể ở cấp độ Hệ thống (Global) để tránh việc Agent bị lẫn lộn ngữ cảnh/dữ liệu khi bạn chuyển đổi giữa các dự án khác nhau.

---

## 3. Các Phương thức Tích hợp GitNexus trên Claude CLI

Claude CLI hỗ trợ cơ chế Hook mạnh mẽ, dẫn đến 3 cách tiếp cận cấu hình khác nhau:

### A. mcpServers-only (Chỉ cấu hình Server)
Claude tự động quản lý vòng đời của GitNexus MCP server.
*   **Ưu điểm:** Cấu hình cực kỳ đơn giản, tự động bật/tắt (auto lifecycle), không cần quản lý tiến trình chạy ngầm.
*   **Nhược điểm:** Thiếu khả năng tự động tối ưu ngữ cảnh (auto-enrich) khi dùng lệnh Grep/Glob thông thường và không tự động cảnh báo index stale (quá hạn).

### B. Hook-based (Chỉ dùng Hooks)
Không khai báo mcpServers, thay vào đó chạy GitNexus mcp làm tiến trình nền (sidecar) và sử dụng Hooks để can thiệp.
*   **Ưu điểm:** Tự động hóa việc "làm giàu" kết quả tìm kiếm (auto-enrich) và tự kiểm tra index freshness.
*   **Nhược điểm:** Phải tự bật/tắt tiến trình nền `gitnexus mcp` bằng tay, dễ bị rò rỉ hoặc treo tiến trình.

### C. Hybrid (mcpServers + Hooks) — Khuyến nghị
Sự kết hợp hoàn hảo: Khai báo `gitnexus` trong danh sách `mcpServers` để Claude tự quản lý vòng đời, đồng thời cấu hình `hooks` để kích hoạt tính năng tự động làm giàu thông tin và kiểm tra index.

---

## 4. Chiến lược Tối ưu hóa khi dùng song song Claude CLI & Antigravity IDE

Khi làm việc đồng thời với cả hai công cụ, chúng ta gặp phải một giới hạn: **Antigravity IDE chỉ hiểu cấu hình `mcpServers` chuẩn, hoàn toàn không hỗ trợ cơ chế `hooks` của Claude.**

Để giải quyết vấn đề này, chiến lược tối ưu nhất là cấu hình **bất đối xứng**:

```mermaid
flowchart LR
  subgraph ClaudeCLI["Claude CLI"]
    direction TB
    C_Config["~/.claude/settings.json"]
    C_Core["Chạy mô hình Hybrid"]
    C_Feature["- Tự quản lý lifecycle<br/>- Tự động enrich context khi Grep/Glob<br/>- Tự động check freshness khi commit"]
    C_Config --> C_Core --> C_Feature
  end
  subgraph AntigravityIDE["Antigravity IDE"]
    direction TB
    I_Config[".mcp.json & mcp_config.json"]
    I_Core["Chạy mô hình mcpServers-only"]
    I_Feature["- Tự quản lý lifecycle<br/>- Sử dụng trực tiếp MCP tools<br/>- Tuân thủ quy trình kiểm tra thủ công qua AGENTS.md"]
    I_Config --> I_Core --> I_Feature
  end

  style ClaudeCLI fill:none,stroke:#047857,stroke-width:2px
  style AntigravityIDE fill:none,stroke:#a21caf,stroke-width:2px
  style C_Config fill:#064e3b,color:#fff,stroke:#047857
  style C_Core fill:#064e3b,color:#fff,stroke:#047857
  style C_Feature fill:#064e3b,color:#fff,stroke:#047857
  style I_Config fill:#701a75,color:#fff,stroke:#a21caf
  style I_Core fill:#701a75,color:#fff,stroke:#a21caf
  style I_Feature fill:#701a75,color:#fff,stroke:#a21caf
```

### Tại sao đây là giải pháp tối ưu?
1.  **Tránh xung đột:** IDE bỏ qua phần `hooks` không được hỗ trợ và chỉ load `mcpServers` bình thường.
2.  **Tự động hóa tối đa trên CLI:** Khi làm việc với Terminal CLI, bạn không cần nhớ chạy các lệnh phân tích thủ công; cơ chế Hook sẽ tự động "nhắc nhở" và cập nhật dữ liệu.
3.  **An toàn tuyệt đối trên IDE:** Agent trên IDE sẽ trực tiếp nhìn thấy các tool `impact` và `detect_changes` trong danh sách MCP Tools và chủ động gọi chúng theo quy chuẩn viết trong file [AGENTS.md](../../../AGENTS.md).

### Vai trò Cầu nối của GitNexus Global
Để cấu hình bất đối xứng này vận hành trơn tru nhất, điểm mấu chốt là **cả Claude CLI và Antigravity IDE đều phải gọi chung bản cài đặt GitNexus Global** (thay vì chạy thông qua `npx` hoặc các phiên bản cục bộ khác nhau). 

Lợi ích của sự nhất quán này bao gồm:
*   **Dùng chung Cơ sở dữ liệu Index:** Cả hai môi trường đều đọc và ghi vào cùng một thư mục dữ liệu phân tích cấu trúc `.gitnexus/` tại root của dự án. Dữ liệu index được tạo ra bởi CLI (ví dụ qua hook sau khi commit) sẽ ngay lập tức được phản ánh và sẵn sàng cho IDE sử dụng mà không cần quét lại.
*   **Tránh xung đột Phiên bản (Version Mismatch):** Nếu Claude CLI dùng một version GitNexus khác (ví dụ qua `npx`) so với IDE, định dạng output của các tool MCP có thể bị lệch cấu trúc. Việc ép cả hai client gọi chung file nhị phân global `/opt/homebrew/bin/gitnexus` đảm bảo hành vi của agent đồng nhất 100%.
*   **Khởi động Tức thì (Zero Startup Latency):** Việc gọi file nhị phân đã cài sẵn trên máy giúp tăng tốc độ load MCP Server khi mở phiên làm việc mới trên cả IDE lẫn CLI, loại bỏ hoàn toàn độ trễ kiểm tra hoặc download từ npm registry của `npx`.
*   **Giải quyết triệt để lỗi môi trường:** Claude CLI khi chạy đôi khi không thừa kế đầy đủ biến môi trường `$PATH` từ shell profile, dẫn đến việc gọi lệnh `gitnexus` trực tiếp bị lỗi. Việc cấu hình cứng đường dẫn tuyệt đối đến file cài đặt global (`/opt/homebrew/bin/gitnexus`) giúp giải quyết hoàn toàn vấn đề này cho cả hai bên.

---

## 5. Hướng dẫn Cấu hình Chi tiết (macOS)

### Bước 1: Cài đặt và Xác minh GitNexus Global
Đảm bảo GitNexus đã được cài đặt trên macOS của bạn để tối ưu tốc độ khởi chạy (tránh sử dụng `npx` tải lại mỗi lần):
```bash
# Cài đặt qua Homebrew (Khuyến nghị trên macOS)
brew install gitnexus

# Hoặc cài đặt qua npm
npm install -g gitnexus
```

**Xác định đường dẫn nhị phân thực tế:**
Chạy lệnh sau trong Terminal để lấy đường dẫn cài đặt chính xác:
```bash
which gitnexus
```
*   **macOS Apple Silicon (M1/M2/M3/M4):** Thường cài đặt tại `/opt/homebrew/bin/gitnexus`
*   **macOS Intel (hoặc cài qua npm global):** Thường cài đặt tại `/usr/local/bin/gitnexus` hoặc `~/.npm-global/bin/gitnexus`

> [!IMPORTANT]
> Hãy copy đường dẫn thực tế trả về từ Terminal để cấu hình vào tham số `"command"` trong các bước bên dưới.

### Bước 2: Thiết lập Hooks cho Claude CLI (Claude Code)
Sao chép các file hook script hỗ trợ phân tích code tự động vào thư mục cấu hình của Claude:
```bash
# Tạo thư mục chứa hooks
mkdir -p ~/.claude/hooks/gitnexus/

# Sao chép hook script từ repo dự án vào thư mục cấu hình global của Claude
cp -r hooks/gitnexus/ ~/.claude/hooks/gitnexus/
```

### Bước 3: Cấu hình `settings.json` cho Claude CLI (Toàn cục)
Chỉnh sửa file cấu hình global của Claude Code tại [settings.json](~/.claude/settings.json) để chạy chế độ **Hybrid** (tự quản lý server và kích hoạt hook):

```json
{
  "model": "dev",
  "mcpServers": {
    "gitnexus": {
      "command": "/opt/homebrew/bin/gitnexus", // Sử dụng đường dẫn thực tế ở Bước 1
      "args": ["mcp"]
    }
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Grep|Glob|Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node '~/.claude/hooks/gitnexus/gitnexus-hook.cjs'",
            "timeout": 10,
            "statusMessage": "Enriching with GitNexus graph context..."
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node '~/.claude/hooks/gitnexus/gitnexus-hook.cjs'",
            "timeout": 10,
            "statusMessage": "Checking GitNexus index freshness..."
          }
        ]
      }
    ]
  },
  "tui": "fullscreen",
  "theme": "auto"
}
```

### Bước 4: Cấu hình MCP cho Antigravity IDE (Gemini)
Bạn có thể chọn một trong hai phương án cấu hình (hoặc cả hai nếu muốn đồng bộ):

#### Phương án A: Cấu hình Cục bộ (Chỉ áp dụng cho dự án hiện tại — Khuyên dùng)
Tạo hoặc chỉnh sửa file [`.mcp.json`](../../../.mcp.json) ở gốc thư mục dự án của bạn:
```json
{
  "mcpServers": {
    "gitnexus": {
      "command": "/opt/homebrew/bin/gitnexus", // Sử dụng đường dẫn thực tế ở Bước 1
      "args": ["mcp"]
    }
  }
}
```

#### Phương án B: Cấu hình Toàn cục (Áp dụng cho mọi dự án mở trên IDE)
Chỉnh sửa file cấu hình global của IDE tại `~/.gemini/config/mcp_config.json` (Đây là file tự động hiển thị khi bạn mở giao diện Settings của IDE):
```json
{
  "mcpServers": {
    "gitnexus": {
      "command": "/opt/homebrew/bin/gitnexus", // Sử dụng đường dẫn thực tế ở Bước 1
      "args": ["mcp"]
    }
  }
}
```

> [!NOTE]
> **Quy tắc ưu tiên:** Nếu file [`.mcp.json`](../../../.mcp.json) cục bộ tồn tại trong thư mục dự án, nó sẽ **ghi đè hoàn toàn** cấu hình của file global `mcp_config.json`.

### Bước 5: Tải lại cấu hình (Reload) để áp dụng thay đổi
*   **Với Claude CLI:** Gõ lệnh `/exit` (hoặc bấm `Ctrl + C`) để thoát phiên làm việc dòng lệnh hiện tại, sau đó khởi chạy lại bằng lệnh `claude`.
*   **Với Antigravity IDE:** Thực hiện reload window của IDE (hoặc khởi động lại ứng dụng) để Agent nạp lại các server MCP mới.

---

## 6. Quy trình làm việc tiêu chuẩn của Nhà phát triển (Developer Workflow)

Khi đã cấu hình cấu trúc bất đối xứng trên, quy trình làm việc hằng ngày của bạn sẽ diễn ra như sau:

```mermaid
sequenceDiagram
    actor Dev as Developer
    participant IDE as Antigravity IDE (Gemini)
    participant CLI as Claude CLI
    participant GN as GitNexus Server

    Note over Dev, CLI: Làm việc trên Terminal CLI
    CLI->>GN: Tự động khởi chạy mcpServer
    Dev->>CLI: Thực hiện tìm kiếm (Grep / Glob)
    CLI->>GN: PreToolUse Hook tự động quét & làm giàu context
    GN-->>CLI: Trả về kết quả kèm file liên quan (Call Graph)
    Dev->>CLI: commit & push code
    CLI->>GN: PostToolUse Hook kiểm tra độ mới của index

    Note over Dev, IDE: Làm việc trên Giao diện IDE
    IDE->>GN: Tự động khởi chạy mcpServer từ file .mcp.json
    Dev->>IDE: Yêu cầu sửa đổi một hàm/class
    Note right of IDE: Agent tự đọc AGENTS.md,<br/>chủ động gọi tool
    IDE->>GN: Gọi tool gitnexus_impact({target: "symbol", direction: "upstream"})
    GN-->>IDE: Trả về blast radius (phạm vi ảnh hưởng trực tiếp)
    Dev->>IDE: Đồng ý chỉnh sửa và tiến hành commit
```

### Quy tắc an toàn bắt buộc khi dev trên IDE:
1.  **Trước khi sửa đổi:** Luôn yêu cầu Agent chạy công cụ `impact` hoặc `context` trên đối tượng chuẩn bị chỉnh sửa.
2.  **Trước khi commit:** Yêu cầu Agent chạy `detect_changes` để so sánh scope thay đổi thực tế với thiết kế ban đầu.
3.  **Sau khi thay đổi lớn:** Nếu có sự thay đổi lớn về cấu trúc file/thư mục, chạy lệnh `node .gitnexus/run.cjs analyze` hoặc `npx gitnexus analyze` từ Terminal để làm tươi (refresh) dữ liệu đồ thị.
