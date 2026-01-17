# Bug 修复指南

## 已自动修复的问题

✅ **HTML 拼写错误**
- 修复了所有文件中的 `shoucut icon` → `shortcut icon`
- 影响文件：
  - [`themes/jasmine/index.html`](themes/jasmine/index.html:12)
  - [`themes/jasmine/article.html`](themes/jasmine/article.html:12)
  - [`themes/jasmine/admin/index.html`](themes/jasmine/admin/index.html:10)
  - [`themes/jasmine/admin/edit.html`](themes/jasmine/admin/edit.html:10)

✅ **编辑器多选分类初始化问题**
- 修复了 [`themes/jasmine/admin/edit.html`](themes/jasmine/admin/edit.html:147-152) 中的多选分类初始化逻辑
- 现在可以正确处理 `category` 为数组或字符串的情况

✅ **日期格式处理优化**
- 修复了 [`themes/jasmine/admin/edit.html`](themes/jasmine/admin/edit.html:138-152) 中的日期格式处理
- 添加了完整的日期格式化和错误处理

✅ **表单 Loading 状态**
- 为所有表单提交添加了 Loading 状态
- 影响文件：
  - [`themes/jasmine/admin/index.html`](themes/jasmine/admin/index.html:325-389) - saveAddNew, saveConfig, importBlog, publish
  - [`themes/jasmine/admin/edit.html`](themes/jasmine/admin/edit.html:155-182) - saveEdit, deleteArticle

---

## 需要手动修复的问题

### 🔴 严重问题

#### 1. 移除硬编码的敏感信息（使用环境变量）

**问题**: [`index.js:3-10`](index.js:3-10) 中硬编码了用户名、密码、API Token 等敏感信息

**修复步骤**:

1. **修改 `wrangler.toml`**，添加环境变量支持:

```toml
name = "cloudflare-workers-blog"
main = "index.js"
compatibility_date = "2026-01-11"

[vars]
siteDomain = "blog.lukelzlz.top"
siteName = "lukelzlz 的博客"
siteDescription = "会一直有人陪你，不会有人一直陪你"
keyWords = "cloudflare,workers,blog,docker,study,php,python,minecraft,linux"
pageSize = 5
recentlySize = 6
readMoreLength = 150
cacheTime = 43200
themeURL = "https://raw.githubusercontent.com/lukelzlz/cloudflare-workers-blog/master/themes/jasmine/"
html404 = "<b>404</b>"
codeBeforHead = ""
codeBeforBody = ""
commentCode = ""
widgetOther = ""
otherCodeA = ""
otherCodeB = ""
otherCodeC = ""
otherCodeD = ""
otherCodeE = ""
copyRight = "©️ lukelzlz 的博客 ｜ 保留所有权利 ｜ Powered by <a href=\"https://www.cloudflare.com\">CF Workers</a> & <a href=\"https://blog.gezhong.vip\">CF-Blog </a>"
robots = "User-agent: *\nDisallow: /admin"

[[kv_namespaces]]
binding = "CFBLOG"
id = "YOUR_KV_NAMESPACE_ID"  # 替换为实际的 KV Namespace ID
```

2. **设置 Secrets**:

```bash
# 设置用户名
wrangler secret put USER
# 输入: lukelzlz

# 设置密码
wrangler secret put PASSWORD
# 输入: lukelzlz2009

# 设置 Cloudflare API Token
wrangler secret put CACHE_TOKEN
# 输入: imoekWOlqPCQ6MKrsuL4kZ9_Zbeand0VcBQ1xPiK

# 设置 Cloudflare Zone ID
wrangler secret put CACHE_ZONE_ID
# 输入: 685f52ed13388befb75f5964002fdd01
```

3. **修改 `index.js`**，使用环境变量:

在 `addEventListener` 之前添加以下代码:

```javascript
// 从环境变量获取配置
const env = {
    USER: typeof USER !== 'undefined' ? USER : "lukelzlz",
    PASSWORD: typeof PASSWORD !== 'undefined' ? PASSWORD : "lukelzlz2009",
    CACHE_TOKEN: typeof CACHE_TOKEN !== 'undefined' ? CACHE_TOKEN : "",
    CACHE_ZONE_ID: typeof CACHE_ZONE_ID !== 'undefined' ? CACHE_ZONE_ID : ""
};

// 更新 OPT 对象
OPT.user = env.USER;
OPT.password = env.PASSWORD;
OPT.cacheToken = env.CACHE_TOKEN;
OPT.cacheZoneId = env.CACHE_ZONE_ID;
```

---

#### 2. KV 命名空间 ID 配置错误

**问题**: [`wrangler.toml:7`](wrangler.toml:7) 中的 KV Namespace ID 是占位符

**修复步骤**:

1. **获取正确的 KV Namespace ID**:

```bash
wrangler kv:namespace list
```

2. **更新 `wrangler.toml`**:

```toml
[[kv_namespaces]]
binding = "CFBLOG"
id = "YOUR_ACTUAL_KV_NAMESPACE_ID"  # 替换为实际的 ID
```

---

#### 3. XSS 安全漏洞（添加内容过滤）

**问题**: 编辑器内容未经过滤直接注入到页面

**修复步骤**:

由于这是一个 Cloudflare Workers 项目，建议在服务端进行内容过滤。在 `index.js` 中添加以下函数:

```javascript
// 添加在 OPT 对象之后
function sanitizeMarkdown(markdown) {
    if (!markdown || typeof markdown !== 'string') return markdown;

    // 移除危险的 HTML 标签和属性
    let sanitized = markdown
        .replace(/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi, '')
        .replace(/<iframe\b[^<]*(?:(?!<\/iframe>)<[^<]*)*<\/iframe>/gi, '')
        .replace(/javascript:/gi, '')
        .replace(/on\w+\s*=/gi, '');

    return sanitized;
}

function sanitizeHTML(html) {
    if (!html || typeof html !== 'string') return html;

    // 移除危险的 HTML 标签和属性
    let sanitized = html
        .replace(/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi, '')
        .replace(/<iframe\b[^<]*(?:(?!<\/iframe>)<[^<]*)*<\/iframe>/gi, '')
        .replace(/javascript:/gi, '')
        .replace(/on\w+\s*=/gi, '')
        .replace(/data:/gi, '');

    return sanitized;
}
```

然后在保存文章时使用这些函数:

```javascript
// 在 saveAddNew 函数中
let t={
    id:y,
    title:sanitizeInput(r),
    img:sanitizeInput(n),
    link:sanitizeInput(a),
    createDate:i,
    category:s.map(sanitizeInput),
    tags:o.split(',').map(sanitizeInput),
    contentMD:sanitizeMarkdown(h),
    contentHtml:sanitizeHTML(d),
    contentText:w,
    priority:u,
    changefreq:g
};
```

---

### 🟠 高优先级问题

#### 4. 添加输入验证和数据清理

**修复步骤**:

在 `index.js` 中添加以下函数:

```javascript
// 输入验证和清理函数
function sanitizeInput(input) {
    if (typeof input !== 'string') return input;

    return input.trim()
        .replace(/[<>]/g, '')  // 移除 HTML 标签字符
        .substring(0, 1000);    // 限制长度
}

function validateURL(url) {
    if (!url) return '';
    try {
        new URL(url);
        return url;
    } catch (e) {
        console.error('Invalid URL:', url);
        return '';
    }
}

function validateDate(dateStr) {
    if (!dateStr) return '';
    try {
        const date = new Date(dateStr);
        if (isNaN(date.getTime())) return '';
        return dateStr;
    } catch (e) {
        console.error('Invalid date:', dateStr);
        return '';
    }
}

function validatePageNumber(page) {
    const pageNum = parseInt(page);
    if (isNaN(pageNum) || pageNum < 1) return 1;
    return pageNum;
}
```

然后在 `saveAddNew` 和 `saveEdit` 函数中使用这些函数:

```javascript
// 示例：在 saveAddNew 中
let r = sanitizeInput(e.title);
let n = validateURL(e.img);
let a = sanitizeInput(e.link);
let i = validateDate(e.createDate);
let s = e.category.map(sanitizeInput);
let o = e.tags.split(',').map(t => sanitizeInput(t.trim())).filter(t => t);
```

---

#### 5. 完善错误处理

**修复步骤**:

在 `index.js` 的主要路由处理函数中添加 try-catch:

```javascript
// 在主处理函数中
try {
    let a = await g("index");
    let s = await l("SYSTEM_VALUE_WidgetMenu", true);
    // ... 其他代码
} catch (error) {
    console.error('Error loading data:', error);
    return new Response('Error loading page', {
        status: 500,
        headers: { 'content-type': 'text/html;charset=UTF-8' }
    });
}
```

---

#### 6. 统一标签数据处理

**修复步骤**:

在 `index.js` 中添加以下函数:

```javascript
// 统一处理标签数据
function normalizeTags(tags) {
    if (Array.isArray(tags)) {
        return tags.map(t => sanitizeInput(t)).filter(t => t);
    }
    if (typeof tags === 'string') {
        return tags.split(',').map(t => sanitizeInput(t.trim())).filter(t => t);
    }
    return [];
}
```

然后在处理标签的地方使用这个函数:

```javascript
// 在 saveAddNew 和 saveEdit 中
let o = normalizeTags(e.tags);
```

---

## 部署步骤

1. **应用所有修复**:
   - 手动修改 `index.js` 和 `wrangler.toml`
   - 设置所有 Secrets

2. **测试本地环境**:
   ```bash
   wrangler dev
   ```

3. **部署到 Cloudflare**:
   ```bash
   wrangler publish
   ```

4. **验证修复**:
   - 检查 favicon 是否正常显示
   - 测试文章编辑和发布
   - 测试表单 Loading 状态
   - 验证环境变量是否正确配置

---

## 安全建议

1. **定期更新依赖**:
   - 定期更新 Bootstrap、Editor.md 等依赖

2. **使用 HTTPS**:
   - 确保所有资源都通过 HTTPS 加载

3. **实施 CSP**:
   - 在 `index.js` 中添加 Content-Security-Policy 头

4. **定期备份**:
   - 定期导出 KV 数据进行备份

5. **监控异常**:
   - 添加日志记录和监控

---

## 后续优化建议

1. **实现搜索功能**
2. **添加文章归档**
3. **实现 RSS 订阅**
4. **优化 SEO**
5. **添加访问统计**
6. **集成评论系统**

---

**最后更新**: 2026-01-17
