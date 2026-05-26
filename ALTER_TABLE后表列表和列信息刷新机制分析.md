# ALTER TABLE 后表列表和列信息刷新机制分析

本文档详细分析 Beekeeper Studio 中执行 ALTER TABLE 后，表列表和列信息的刷新机制。

## 一、updateTables 为什么只对已有列缓存的表重新拉列

### 核心逻辑位置
[store/index.ts#L630-L668](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/index.ts#L630-L668)

### 实现细节

```typescript
async updateTables(context) {
  try {
    // ... 拉取表列表代码 ...
    
    // 关键逻辑：只对已有列缓存的表重新拉取列
    for (const table of tables) {
      const match = context.state.tables.find((st) => tablesMatch(st, table))
      if (match?.columns?.length > 0) {
        table.columns = (table.entityType === 'materialized-view' ?
          await context.state.connection?.listMaterializedViewColumns(table.name, table.schema) :
          await context.state.connection?.listTableColumns(table.name, table.schema)) || []
      }
    }
    
    context.commit('tables', tables)
  } finally {
    context.commit("tablesLoading", null)
  }
}
```

### 设计原因

1. **性能优化**：
   - 数据库中可能有数百甚至数千张表
   - 如果对所有表都拉取列信息，会产生大量数据库查询，严重影响性能
   - 只对已经缓存过列的表刷新，避免了不必要的数据库查询

2. **用户体验**：
   - 用户未展开过的表不需要列信息
   - 只刷新用户关心的（已展开过的）表的列，减少加载时间

3. **按需加载策略**：
   - 列信息采用懒加载模式，只有当用户真正需要时才加载
   - 表列表刷新时保持这种策略的一致性

## 二、tables mutation 如何保留对象引用

### 核心逻辑位置
[store/index.ts#L383-L404](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/index.ts#L383-L404)

### 实现细节

```typescript
tables(state, tables: TableOrView[]) {
  if(state.tables.length === 0) {
    state.tables = tables
  } else {
    const result = tables.map((t) => {
      t.columns ||= []
      const existingIdx = state.tables.findIndex((st) => tablesMatch(st, t))
      if (existingIdx >= 0) {
        const existing = state.tables[existingIdx]
        // 关键：使用 Object.assign 合并属性，保留原对象引用
        Object.assign(existing, t)
        return existing
      } else {
        return t
      }
    })
    state.tables = result
  }
  
  if (!state.tablesInitialLoaded) state.tablesInitialLoaded = true
}
```

### 表匹配逻辑
[store/index.ts#L46-L50](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/index.ts#L46-L50)

```typescript
const tablesMatch = (t: TableOrView, t2: TableOrView) => {
  return t2.name === t.name &&
    t2.schema === t.schema &&
    t2.entityType === t.entityType
}
```

### 保留对象引用的好处

1. **Vue 响应式优化**：
   - Vue 的响应式系统依赖于对象引用
   - 如果每次都创建新对象，会导致所有依赖该对象的组件重新渲染
   - 保留引用可以避免不必要的重渲染

2. **组件状态保持**：
   - Sidebar 中的展开/收起状态是绑定在表对象上的
   - 如果对象引用变化，展开状态会丢失
   - 保留引用可以维持用户的 UI 操作状态

3. **内存效率**：
   - 避免频繁创建和销毁对象
   - 减少 GC 压力

### 相关设计模式
[StoreHelpers.ts#L55-L63](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/StoreHelpers.ts#L55-L63) 中的 `upsert` 函数也采用了相同的模式：

```typescript
export function upsert<T extends HasId>(list: T[], item: T, func = (a: T, b: T) => a.id === b.id) {
  const found = _.find(list, (q: T) => func(q, item))
  if (found) {
    Object.assign(found, item)  // 保留引用，合并属性
  } else {
    list.push(item)
  }
}
```

## 三、sidebar 虚拟列表何时懒加载列

### 核心逻辑位置
[VirtualTableList.vue](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/sidebar/core/table_list/VirtualTableList.vue)

### 懒加载触发时机

#### 1. 用户点击展开表时
[VirtualTableList.vue#L242-L248](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/sidebar/core/table_list/VirtualTableList.vue#L242-L248)

```typescript
handleExpand(_: Event, item: Item) {
  item.expanded = !item.expanded;
  if (item.expanded && item.type === "table") {
    this.loadColumns(item);  // 展开时加载列
  }
  this.generateDisplayItems();
}
```

#### 2. 滚动结束时加载可视范围内的列
[VirtualTableList.vue#L286-L288](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/sidebar/core/table_list/VirtualTableList.vue#L286-L288)

```typescript
handleScrollEnd() {
  this.updateTableColumnsInRange(true);  // 只加载没有列的表
}
```

#### 3. 全部展开时加载可视范围内的列
[VirtualTableList.vue#L263-L276](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/sidebar/core/table_list/VirtualTableList.vue#L263-L276)

```typescript
handleToggleExpandedAll(expand?: boolean) {
  if (typeof expand === "undefined") {
    expand = false;
  }
  this.items.forEach((item: Item) => {
    item.expanded = expand;
  });
  this.generateDisplayItems();
  if (expand) {
    this.$nextTick(() => {
      this.updateTableColumnsInRange();  // 全部展开后加载可视范围的列
    });
  }
}
```

### 可视范围列加载逻辑
[VirtualTableList.vue#L229-L241](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/sidebar/core/table_list/VirtualTableList.vue#L229-L241)

```typescript
updateTableColumnsInRange(whenEmpty = false) {
  const range = this.$refs.vList.range;
  
  for (let i = range.start; i < range.end + 1; i++) {
    const item = this.displayItems[i];
    if (!item.expanded) continue;           // 只处理已展开的
    if (item.type !== "table") continue;    // 只处理表
    if (whenEmpty && item.entity.columns?.length) continue;  // 可选：只加载没有列的
    if (item.loadingColumns) continue;      // 避免重复加载
    
    this.loadColumns(item);
  }
}
```

### 列加载实现
[VirtualTableList.vue#L221-L228](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/sidebar/core/table_list/VirtualTableList.vue#L221-L228)

```typescript
loadColumns(item: TableItem) {
  item.loadingColumns = true;
  this.$nextTick(() => {
    this.$store.dispatch("updateTableColumns", item.entity).finally(() => {
      item.loadingColumns = false;
    });
  });
}
```

### 虚拟列表设计背景
组件注释说明了为什么需要虚拟列表：

> In some cases, there are databases that have over 1000 tables or even much
> more. Without optimizing the table list, it can be very slow.
> 
> This component will render the items that are visible. Items that are outside
> the viewport are destroyed. So all the states that need to be persistent
> should be stored here instead of the item component.

## 四、插件调用 getColumns 时如何补齐缺失列

### 核心逻辑位置
[PluginStoreService.ts#L223-L243](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/services/plugin/web/PluginStoreService.ts#L223-L243)

### 实现细节

```typescript
async getColumns(
  tableName: string,
  schema?: string
) {
  // 1. 查找表对象
  const table = this.findTableOrThrow(tableName, schema);

  // 2. 检查列是否已缓存，如果没有则加载
  if (!table.columns || table.columns.length === 0) {
    await this.store.dispatch("updateTableColumns", table);
  }

  // 3. 返回列信息（再次查找确保获取最新数据）
  return this.findTable(tableName, schema).columns.map((c: ExtendedTableColumn) => ({
    name: c.columnName,
    type: c.dataType,
    comment: c.comment ?? "",
    nullable: c.nullable ?? false,
    defaultValue: c.defaultValue,
    extra: c.extra ?? "",
    generated: c.generated ?? false,
    ordinalPosition: c.ordinalPosition,
  }));
}
```

### 表查找逻辑
[PluginStoreService.ts#L202-L212](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/services/plugin/web/PluginStoreService.ts#L202-L212)

```typescript
private findTable(name: string, schema?: string) {
  return this.store.state.tables.find((t) => {
    if (!schema && this.store.state.defaultSchema) {
      schema = this.store.state.defaultSchema;
    }
    if (schema) {
      return t.name === name && t.schema === schema;
    }
    return t.name === name;
  });
}
```

### updateTableColumns action
[store/index.ts#L599-L618](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/index.ts#L599-L618)

```typescript
async updateTableColumns(context, table: TableOrView) {
  log.debug('actions/updateTableColumns', table.name)
  try {
    context.commit("columnsLoading", "Loading columns...")
    const columns = (table.entityType === 'materialized-view' ?
        await context.state.connection.listMaterializedViewColumns(table.name, table.schema) :
        await context.state.connection.listTableColumns(table.name, table.schema));

    const updated = _.xorWith(table.columns, columns, _.isEqual)
    log.debug('Should I update table columns?', updated)
    if (updated?.length) {
      context.commit('table', { ...table, columns })
    }
  } finally {
    context.commit("columnsLoading", null)
  }
}
```

### 设计要点

1. **自动补齐**：插件不需要关心列是否已加载，`getColumns` 会自动处理
2. **缓存复用**：如果列已经加载，直接返回缓存数据
3. **数据一致性**：通过 `_.xorWith` 比较列差异，只有真正变化时才更新 store
4. **类型转换**：将内部的 `ExtendedTableColumn` 转换为插件使用的简化格式

## 五、整体刷新流程总结

执行 ALTER TABLE 后，表列表和列信息的刷新流程如下：

```
用户执行 ALTER TABLE
        ↓
触发 updateTables action
        ↓
1. 拉取最新表列表（listTables + listViews + listMaterializedViews）
2. 遍历所有表，对已有列缓存的表重新拉取列信息
        ↓
提交 tables mutation
        ↓
1. 如果是首次加载，直接赋值
2. 否则，通过 Object.assign 保留对象引用，合并新属性
        ↓
Sidebar 虚拟列表响应式更新
        ↓
1. 未展开的表：不加载列，保持懒加载
2. 已展开且在可视范围：滚动结束时自动加载列
3. 用户点击展开：即时加载该表的列
        ↓
插件调用 getColumns 时
        ↓
1. 检查列是否已缓存
2. 未缓存则调用 updateTableColumns 加载
3. 返回格式化后的列信息
```

## 六、关键文件索引

| 功能 | 文件路径 |
|------|---------|
| updateTables action | [store/index.ts#L630-L668](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/index.ts#L630-L668) |
| tables mutation | [store/index.ts#L383-L404](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/index.ts#L383-L404) |
| 虚拟列表组件 | [VirtualTableList.vue](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/sidebar/core/table_list/VirtualTableList.vue) |
| 插件 getColumns | [PluginStoreService.ts#L223-L243](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/services/plugin/web/PluginStoreService.ts#L223-L243) |
| updateTableColumns action | [store/index.ts#L599-L618](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/index.ts#L599-L618) |
| upsert 工具函数 | [StoreHelpers.ts#L55-L63](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/StoreHelpers.ts#L55-L63) |
