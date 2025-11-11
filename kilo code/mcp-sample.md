# call-tools.md

# RULES GỌI MCP TOOLS - KHÔNG BAO GIỜ LỖI

## 📌 QUY TẮC VÀNG

**MỌI LỜI GỌI MCP TOOL PHẢI BỌC TRONG `<use_mcp_tool>`**

## 🔄 CẤU TRÚC BẮT BUỘC

```xml
<use_mcp_tool>
<server_name>TÊN_SERVER</server_name>
<tool_name>TÊN_TOOL_CON</tool_name>
<arguments>{JSON_HỢP_LỆ}</arguments>
</use_mcp_tool>
```

## ✅ MẪU ĐÚNG - COPY PASTE ĐI

```xml
<use_mcp_tool>
<server_name>exa</server_name>
<tool_name>web_search_exa</tool_name>
<arguments>
{
  "query": "tìm kiếm gì đó",
  "numResults": 5
}
</arguments>
</use_mcp_tool>
```

## ❌ MẪU SAI - TUYỆT ĐỐI KHÔNG DÙNG

```xml
<web_search_exa>
<query>tìm kiếm</query>
</web_search_exa>
```

## 📋 CHECKLIST 3 BƯỚC

1. **Bọc trong `<use_mcp_tool>`?** ✓
2. **Có đủ 3 tham số: server_name, tool_name, arguments?** ✓  
3. **Arguments là JSON hợp lệ?** ✓

## 🎯 QUY TẮC ĐƠN GIẢN

> **KHÔNG BAO GIỜ GỌI TRỰC TIẾP TÊN TOOL MCP**
> 
> **LUÔN LUÔN BỌC TRONG `<use_mcp_tool>`**

Chỉ cần nhớ quy tắc này là không bao giờ lỗi!