# Work It Out - 数据库 Schema 设计

## 概述
本文档定义了 Work It Out 应用的完整数据库结构，所有模块必须严格遵循此 Schema。

**数据库**: PostgreSQL 15+
**编码**: UTF-8
**时区**: UTC

---

## 1. 用户相关表

### 1.1 users - 用户表
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    avatar_url VARCHAR(500),
    institution VARCHAR(200),          -- 所属机构
    bio TEXT,                          -- 个人简介
    is_active BOOLEAN DEFAULT TRUE,
    is_verified BOOLEAN DEFAULT FALSE, -- 邮箱是否验证
    last_login_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_is_active ON users(is_active);
```

### 1.2 refresh_tokens - 刷新令牌表
```sql
CREATE TABLE refresh_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL UNIQUE,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    revoked_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_refresh_tokens_user ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_hash ON refresh_tokens(token_hash);
```

---

## 2. 项目相关表

### 2.1 projects - 项目表
```sql
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(500) NOT NULL,
    description TEXT,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'completed', 'archived')),
    template_type VARCHAR(50),         -- 模板类型: undergraduate, master, phd, journal, custom
    target_word_count INTEGER,         -- 目标字数
    current_word_count INTEGER DEFAULT 0,
    start_date DATE,
    target_date DATE,                  -- 目标完成日期
    completed_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_projects_owner ON projects(owner_id);
CREATE INDEX idx_projects_status ON projects(status);
```

### 2.2 project_members - 项目成员表
```sql
CREATE TABLE project_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL CHECK (role IN ('owner', 'collaborator', 'advisor', 'reviewer')),
    permissions JSONB DEFAULT '{"can_edit": true, "can_comment": true, "can_manage": false}',
    invited_by UUID REFERENCES users(id),
    joined_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(project_id, user_id)
);

CREATE INDEX idx_project_members_project ON project_members(project_id);
CREATE INDEX idx_project_members_user ON project_members(user_id);
```

---

## 3. 任务相关表

### 3.1 tasks - 任务表
```sql
CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    parent_id UUID REFERENCES tasks(id) ON DELETE CASCADE,  -- 支持子任务
    title VARCHAR(500) NOT NULL,
    description TEXT,
    status VARCHAR(20) DEFAULT 'todo' CHECK (status IN ('todo', 'in_progress', 'completed')),
    priority VARCHAR(10) DEFAULT 'medium' CHECK (priority IN ('low', 'medium', 'high')),
    phase VARCHAR(50),                 -- 论文阶段: topic_selection, literature_review, methodology, data_collection, analysis, writing, revision
    assignee_id UUID REFERENCES users(id) ON DELETE SET NULL,
    start_date DATE,
    due_date DATE,
    completed_at TIMESTAMP WITH TIME ZONE,
    position INTEGER DEFAULT 0,        -- 排序位置
    estimated_hours DECIMAL(5,2),      -- 预估工时
    actual_hours DECIMAL(5,2),         -- 实际工时
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_tasks_project ON tasks(project_id);
CREATE INDEX idx_tasks_parent ON tasks(parent_id);
CREATE INDEX idx_tasks_assignee ON tasks(assignee_id);
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_due_date ON tasks(due_date);
```

### 3.2 milestones - 里程碑表
```sql
CREATE TABLE milestones (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    due_date DATE NOT NULL,
    is_completed BOOLEAN DEFAULT FALSE,
    completed_at TIMESTAMP WITH TIME ZONE,
    position INTEGER DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_milestones_project ON milestones(project_id);
CREATE INDEX idx_milestones_due_date ON milestones(due_date);
```

### 3.3 task_labels - 任务标签表
```sql
CREATE TABLE task_labels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name VARCHAR(50) NOT NULL,
    color VARCHAR(7) DEFAULT '#3B82F6',  -- HEX 颜色
    UNIQUE(project_id, name)
);

CREATE TABLE task_label_relations (
    task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    label_id UUID NOT NULL REFERENCES task_labels(id) ON DELETE CASCADE,
    PRIMARY KEY(task_id, label_id)
);
```

---

## 4. 文献相关表

### 4.1 literature - 文献表
```sql
CREATE TABLE literature (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    title VARCHAR(1000) NOT NULL,
    authors JSONB,                     -- [{"name": "张三", "affiliation": "北京大学"}]
    year INTEGER,
    publication_type VARCHAR(50),      -- journal, conference, book, thesis, website, other
    journal VARCHAR(500),              -- 期刊/会议名称
    volume VARCHAR(50),
    issue VARCHAR(50),
    pages VARCHAR(50),
    publisher VARCHAR(200),
    doi VARCHAR(255),
    isbn VARCHAR(20),
    url VARCHAR(1000),
    abstract TEXT,
    keywords JSONB,                    -- ["关键词1", "关键词2"]
    bibtex TEXT,                       -- 原始 BibTeX
    citation_key VARCHAR(100),         -- 引用键，如 "zhang2024"
    pdf_path VARCHAR(500),             -- PDF 文件路径
    read_status VARCHAR(20) DEFAULT 'unread' CHECK (read_status IN ('unread', 'reading', 'completed')),
    rating INTEGER CHECK (rating >= 1 AND rating <= 5),
    added_by UUID REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_literature_project ON literature(project_id);
CREATE INDEX idx_literature_doi ON literature(doi);
CREATE INDEX idx_literature_citation_key ON literature(citation_key);
CREATE INDEX idx_literature_read_status ON literature(read_status);
```

### 4.2 literature_tags - 文献标签表
```sql
CREATE TABLE literature_tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    color VARCHAR(7) DEFAULT '#10B981',
    UNIQUE(project_id, name)
);

CREATE TABLE literature_tag_relations (
    literature_id UUID NOT NULL REFERENCES literature(id) ON DELETE CASCADE,
    tag_id UUID NOT NULL REFERENCES literature_tags(id) ON DELETE CASCADE,
    PRIMARY KEY(literature_id, tag_id)
);

CREATE INDEX idx_lit_tag_relations_literature ON literature_tag_relations(literature_id);
CREATE INDEX idx_lit_tag_relations_tag ON literature_tag_relations(tag_id);
```

### 4.3 literature_notes - 文献笔记表
```sql
CREATE TABLE literature_notes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    literature_id UUID NOT NULL REFERENCES literature(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    note_type VARCHAR(20) DEFAULT 'general' CHECK (note_type IN ('general', 'highlight', 'question', 'idea')),
    page_number INTEGER,               -- PDF 页码
    highlight_text TEXT,               -- 高亮的原文
    position_data JSONB,               -- PDF 中的位置信息
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_literature_notes_literature ON literature_notes(literature_id);
CREATE INDEX idx_literature_notes_user ON literature_notes(user_id);
```

---

## 5. 文档相关表

### 5.1 documents - 文档表
```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    parent_id UUID REFERENCES documents(id) ON DELETE CASCADE,  -- 支持章节层级
    title VARCHAR(500) NOT NULL,
    content TEXT,                      -- 文档内容 (JSON 格式，Tiptap 编辑器)
    content_text TEXT,                 -- 纯文本内容，用于搜索
    doc_type VARCHAR(20) DEFAULT 'section' CHECK (doc_type IN ('folder', 'section', 'appendix')),
    position INTEGER DEFAULT 0,        -- 同级排序
    word_count INTEGER DEFAULT 0,
    is_locked BOOLEAN DEFAULT FALSE,   -- 是否锁定编辑
    locked_by UUID REFERENCES users(id),
    locked_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_documents_project ON documents(project_id);
CREATE INDEX idx_documents_parent ON documents(parent_id);
CREATE INDEX idx_documents_position ON documents(project_id, parent_id, position);
```

### 5.2 document_versions - 文档版本表
```sql
CREATE TABLE document_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    version_number INTEGER NOT NULL,
    content TEXT NOT NULL,             -- 版本内容快照
    content_text TEXT,
    word_count INTEGER DEFAULT 0,
    change_summary VARCHAR(500),       -- 版本说明
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(document_id, version_number)
);

CREATE INDEX idx_document_versions_document ON document_versions(document_id);
CREATE INDEX idx_document_versions_created_at ON document_versions(created_at);
```

### 5.3 document_citations - 文档引用关联表
```sql
CREATE TABLE document_citations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    literature_id UUID NOT NULL REFERENCES literature(id) ON DELETE CASCADE,
    citation_key VARCHAR(100),
    position INTEGER,                  -- 在文档中的位置
    context TEXT,                      -- 引用上下文
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_document_citations_document ON document_citations(document_id);
CREATE INDEX idx_document_citations_literature ON document_citations(literature_id);
```

---

## 6. 协作相关表

### 6.1 comments - 评论表
```sql
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    version_id UUID REFERENCES document_versions(id) ON DELETE SET NULL,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    parent_id UUID REFERENCES comments(id) ON DELETE CASCADE,  -- 支持回复
    content TEXT NOT NULL,
    selection_start INTEGER,           -- 选中文本起始位置
    selection_end INTEGER,             -- 选中文本结束位置
    selected_text TEXT,                -- 选中的文本内容
    is_resolved BOOLEAN DEFAULT FALSE,
    resolved_by UUID REFERENCES users(id),
    resolved_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_comments_document ON comments(document_id);
CREATE INDEX idx_comments_user ON comments(user_id);
CREATE INDEX idx_comments_parent ON comments(parent_id);
CREATE INDEX idx_comments_resolved ON comments(is_resolved);
```

### 6.2 notifications - 通知表
```sql
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type VARCHAR(50) NOT NULL,         -- task_due, task_assigned, comment_added, mention, milestone_due, project_invite
    title VARCHAR(255) NOT NULL,
    content TEXT,
    link VARCHAR(500),                 -- 跳转链接
    related_id UUID,                   -- 关联的实体 ID
    related_type VARCHAR(50),          -- 关联实体类型: task, comment, project, etc.
    is_read BOOLEAN DEFAULT FALSE,
    read_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_notifications_user ON notifications(user_id);
CREATE INDEX idx_notifications_read ON notifications(user_id, is_read);
CREATE INDEX idx_notifications_created ON notifications(created_at);
```

### 6.3 activity_logs - 活动日志表
```sql
CREATE TABLE activity_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    action VARCHAR(50) NOT NULL,       -- created, updated, deleted, completed, commented, etc.
    entity_type VARCHAR(50) NOT NULL,  -- project, task, document, literature, comment
    entity_id UUID NOT NULL,
    entity_title VARCHAR(500),
    details JSONB,                     -- 变更详情
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_activity_logs_project ON activity_logs(project_id);
CREATE INDEX idx_activity_logs_user ON activity_logs(user_id);
CREATE INDEX idx_activity_logs_created ON activity_logs(created_at);
```

---

## 7. 系统表

### 7.1 file_uploads - 文件上传表
```sql
CREATE TABLE file_uploads (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    original_name VARCHAR(255) NOT NULL,
    stored_name VARCHAR(255) NOT NULL,
    file_path VARCHAR(500) NOT NULL,
    file_size BIGINT NOT NULL,
    mime_type VARCHAR(100),
    file_type VARCHAR(50),             -- avatar, pdf, image, attachment
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_file_uploads_user ON file_uploads(user_id);
CREATE INDEX idx_file_uploads_project ON file_uploads(project_id);
```

---

## 8. 触发器和函数

### 8.1 自动更新 updated_at
```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- 为所有需要的表创建触发器
CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_projects_updated_at BEFORE UPDATE ON projects
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_tasks_updated_at BEFORE UPDATE ON tasks
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_milestones_updated_at BEFORE UPDATE ON milestones
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_literature_updated_at BEFORE UPDATE ON literature
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_literature_notes_updated_at BEFORE UPDATE ON literature_notes
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_documents_updated_at BEFORE UPDATE ON documents
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_comments_updated_at BEFORE UPDATE ON comments
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### 8.2 自动计算项目字数
```sql
CREATE OR REPLACE FUNCTION update_project_word_count()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE projects
    SET current_word_count = (
        SELECT COALESCE(SUM(word_count), 0)
        FROM documents
        WHERE project_id = COALESCE(NEW.project_id, OLD.project_id)
    )
    WHERE id = COALESCE(NEW.project_id, OLD.project_id);
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_project_word_count_trigger
AFTER INSERT OR UPDATE OR DELETE ON documents
    FOR EACH ROW EXECUTE FUNCTION update_project_word_count();
```

---

## 9. 初始数据

### 9.1 默认任务阶段
```sql
-- 可以在应用层定义，不需要数据库表
-- 阶段列表:
-- topic_selection: 选题
-- literature_review: 文献综述
-- methodology: 研究方法
-- data_collection: 数据收集
-- analysis: 数据分析
-- writing: 论文撰写
-- revision: 修改完善
-- submission: 提交答辩
```

---

## 10. 数据库配置建议

### 10.1 连接池配置
```python
# SQLAlchemy 配置
SQLALCHEMY_DATABASE_URL = "postgresql+asyncpg://user:password@localhost:5432/workitout"

# 连接池设置
POOL_SIZE = 20
MAX_OVERFLOW = 10
POOL_TIMEOUT = 30
POOL_RECYCLE = 1800
```

### 10.2 索引优化建议
- 对于频繁查询的字段已创建索引
- 复合索引用于常见的多条件查询
- JSONB 字段可根据需要添加 GIN 索引

### 10.3 分区建议（数据量大时）
- `activity_logs` 表可按月分区
- `notifications` 表可按月分区
- `document_versions` 表可按项目分区
