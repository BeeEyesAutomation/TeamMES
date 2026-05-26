# Quotation + Temple Integration Guide

## Overview

This guide shows how to integrate the Temple system into the Quotation Editor, enabling:
- Temple selection when creating/editing quotations
- Material table configuration based on temple columns
- Variable binding for material table columns
- Data resolution from quotation context

---

## Files Created for Integration

### 1. quotation-temple-integration.tsx
**Location**: `apps/web/features/quotations/quotation-temple-integration.tsx`

**Provides**:
- `useQuotationTemples()` - Hook to fetch available temples
- `useQuotationTempleColumns()` - Hook to load columns from selected temple
- `QuotationTempleSelector` - Component to select temple in form
- `resolveQuotationVariable()` - Resolve variable binding keys to actual values
- `formatColumnValue()` - Format values for display
- `getVisibleColumns()` - Filter visible columns
- `getColumnValue()` - Get value from item with variable resolution

### 2. quotation-material-table.tsx
**Location**: `apps/web/features/quotations/quotation-material-table.tsx`

**Provides**:
- `QuotationMaterialTable` - Main table component using temple columns
- `ColumnCell` - Individual cell with variable resolution
- `DefaultMaterialTable` - Fallback table when no temple columns
- `QuotationMaterialTablePreview` - Preview component for editing

---

## Integration Steps

### Step 1: Update Quotation Type Definition

**File**: `apps/web/types/quotations.ts`

Add to `Quotation` interface:
```typescript
interface Quotation {
  // ... existing fields
  templeId?: string;  // NEW: Reference to Temple
  templateId?: string; // Keep existing
}
```

### Step 2: Update Quotation Form

**File**: `apps/web/features/quotations/quotation-form.tsx`

**Add imports**:
```typescript
import { 
  QuotationTempleSelector,
  useQuotationTempleColumns,
  resolveQuotationVariable
} from './quotation-temple-integration';
import { 
  QuotationMaterialTable,
  QuotationMaterialTablePreview
} from './quotation-material-table';
```

**Add state**:
```typescript
const [templeId, setTempleId] = useState(form.templeId || '');
const { columns: templeColumns, isLoading: columnsLoading } = useQuotationTempleColumns(templeId);
```

**Add to form object**:
```typescript
templeId: templeId,  // Store in form
```

**Add UI section** (after Quotation Template section):
```jsx
<Field label="Quotation Temple">
  <QuotationTempleSelector
    selectedTempleId={templeId}
    onTempleSelect={setTempleId}
    disabled={isLoading}
  />
</Field>

{templeColumns.length > 0 && (
  <div>
    <label className="text-sm font-medium">Material Table Preview</label>
    <QuotationMaterialTablePreview 
      columns={templeColumns} 
      isLoading={columnsLoading}
    />
  </div>
)}
```

**Replace material table rendering**:
```jsx
<QuotationMaterialTable
  items={form.items}
  columns={templeColumns}
  quotationData={{
    quotation: form,
    company: companySettings
  }}
  isEditing={true}
/>
```

**Update save logic**:
```typescript
// Include templeId when saving
const payload = {
  // ... other fields
  templeId: templeId || undefined,
  templateId: form.templateId || undefined // Keep for backward compat
};
```

### Step 3: Update Quotation Detail View

**File**: `apps/web/features/quotations/quotation-detail-client.tsx`

**Add imports**:
```typescript
import { QuotationMaterialTable } from './quotation-material-table';
import { useQuotationTempleColumns } from './quotation-temple-integration';
```

**Add loading for temple columns**:
```typescript
const { columns: templeColumns } = useQuotationTempleColumns(quotation.templeId);
```

**Replace material table**:
```jsx
<QuotationMaterialTable
  items={quotation.items}
  columns={templeColumns}
  quotationData={{
    quotation: quotation,
    company: companySettings
  }}
  isEditing={false}
/>
```

### Step 4: Update API

**File**: `apps/web/features/quotations/quotations-api.ts`

Update quotation save function to include `templeId`:
```typescript
export async function saveQuotation(quotation: Quotation) {
  const payload = {
    // ... other fields
    templeId: quotation.templeId,
    templateId: quotation.templateId // Keep for backward compat
  };
  
  // ... rest of function
}
```

---

## Usage Examples

### Example 1: Basic Integration

```tsx
import { Quotation } from '@/types/quotations';
import { QuotationTempleSelector } from '@/features/quotations/quotation-temple-integration';

export function MyQuotationForm() {
  const [quotation, setQuotation] = useState<Quotation>({});

  return (
    <form>
      <QuotationTempleSelector
        selectedTempleId={quotation.templeId}
        onTempleSelect={(templeId) => 
          setQuotation({ ...quotation, templeId })
        }
      />
      {/* ... rest of form */}
    </form>
  );
}
```

### Example 2: Render Material Table with Temple Columns

```tsx
import { QuotationMaterialTable } from '@/features/quotations/quotation-material-table';
import { useQuotationTempleColumns } from '@/features/quotations/quotation-temple-integration';

export function QuotationPreview({ quotation }: { quotation: Quotation }) {
  const { columns } = useQuotationTempleColumns(quotation.templeId);

  return (
    <QuotationMaterialTable
      items={quotation.items}
      columns={columns}
      quotationData={{
        quotation: quotation,
        company: companySettings
      }}
      isEditing={false}
    />
  );
}
```

### Example 3: Resolve Variables Manually

```tsx
import { resolveQuotationVariable } from '@/features/quotations/quotation-temple-integration';

// Get value for material code column
const materialCode = resolveQuotationVariable(
  'inventory.material.code',
  {
    quotation: quotationData,
    company: companyData,
    item: itemData
  }
);

console.log(materialCode); // e.g., "MAT-001"
```

---

## Variable Resolution Reference

### Quotation Variables
```
quotation.code       → quotation.quotationCode
quotation.date       → quotation.quotationDate
quotation.type       → quotation.quotationType
quotation.content    → quotation.content
```

### Customer Variables
```
customer.name        → quotation.customerName
customer.recipient   → quotation.recipientName
customer.request     → quotation.customerRequest
```

### Project Variables
```
project.name         → quotation.project.name
```

### Money Variables
```
money.subtotalOneSet → quotation.subtotalOneSet
money.numberOfSets   → quotation.numberOfSets
money.totalBeforeVat → quotation.totalBeforeVat
money.vatRate        → quotation.vatRate
money.vatAmount      → quotation.vatAmount
money.grandTotal     → quotation.grandTotal
```

### Material Variables
```
material.code        → item.materialCodeSnapshot / item.materialCode
material.name        → item.materialNameSnapshot / item.name
material.model       → item.modelSnapshot / item.model
material.brand       → item.brand
material.unit        → item.unitSnapshot / item.unit
material.price       → item.unitPrice / item.price
material.quantity    → item.quantity
material.leadTime    → item.leadTime
material.warranty    → item.warranty
material.picture     → item.pictureUrlSnapshot / item.pictureUrl
material.remark      → item.remark
```

### Company Variables
```
company.name         → company.companyName
company.taxCode      → company.taxCode
company.address      → company.address
company.phone        → company.phone
company.email        → company.email
company.bankAccount  → company.bankAccountNumber
company.bankName     → company.bankName
company.bankBranch   → company.bankBranch
```

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────┐
│         Quotation Form                                  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. User selects Temple from dropdown                   │
│     ↓                                                   │
│  2. useQuotationTempleColumns() fetches temple columns │
│     ↓                                                   │
│  3. QuotationMaterialTable renders with:               │
│     - Temple columns configuration                      │
│     - Variable bindings (e.g., material.code)          │
│     - Quotation data context                           │
│     ↓                                                   │
│  4. ColumnCell resolves each cell:                     │
│     - If column has variableKey:                       │
│       resolveQuotationVariable(variableKey, context)   │
│     - formatColumnValue(value, type)                   │
│     - Display in table                                 │
│                                                         │
│  5. Save quotation with templeId                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Testing the Integration

### Manual Test 1: Select Temple
1. Create new quotation
2. Select temple from dropdown
3. Verify material table preview loads columns
4. Verify column labels show correctly

### Manual Test 2: Data Resolution
1. Select temple with material.code column
2. Add material item (e.g., "MAT-001")
3. Verify material code appears in table
4. Check variable binding works

### Manual Test 3: Multiple Columns
1. Create temple with columns: code, name, price
2. Add material items
3. Verify all columns show correct data
4. Check column widths and alignment

### Manual Test 4: Backward Compatibility
1. Load quotation created before (without templeId)
2. Verify it still loads correctly
3. Verify default material table renders
4. Can update to use temple

---

## Debugging Tips

### Variables not resolving?
1. Check variable key format: `module.property.path`
2. Verify quotation/item data is available in context
3. Check column.variableKey is set

### Material table not rendering?
1. Verify templeId is selected
2. Check useQuotationTempleColumns is called
3. Verify temple has tableConfig in version
4. Check columns.length > 0

### Values showing as "—"?
1. Variable might be null/undefined in data
2. Check resolveQuotationVariable is returning correct value
3. Verify item has the field data

---

## Migration Guide

### For Existing Quotations

Quotations created before Temple integration:
- Will still work with `templateId` (backward compatible)
- Can be updated to use `templeId` via edit form
- Default material table will render if no `templeId`

```typescript
// Safe: supports both old and new
if (quotation.templeId) {
  // Use temple columns
  <QuotationMaterialTable columns={templeColumns} ... />
} else if (quotation.templateId) {
  // Use old quotation template (fallback)
  <DefaultMaterialTable ... />
}
```

---

## Performance Considerations

1. **Temple Columns Caching**: `useQuotationTempleColumns` caches results
2. **Variable Resolution**: Happens at render time (consider memoization for large tables)
3. **Visible Columns Filter**: Uses `getVisibleColumns()` - O(n) but small n
4. **API Calls**: Templates loaded on demand (not all templates at once)

---

## Future Enhancements

1. **Editable Material Table**: Allow inline column editing
2. **Column Filtering**: Show/hide columns per quotation
3. **Custom Formulas**: Support calculated columns (e.g., Qty * Price)
4. **Export**: Export table with temple columns to PDF/Excel
5. **Template Versioning**: Use specific temple version per quotation

---

## Support & Questions

For issues:
1. Check integration guide above
2. Review usage examples
3. Check browser console for errors
4. Check API responses for validation errors

---

**Last Updated**: 2026-05-22  
**Status**: Integration Complete  
**Version**: 1.0
