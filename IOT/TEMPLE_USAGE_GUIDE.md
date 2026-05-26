# Temple Implementation - Usage Guide

## 🎯 Overview

The Temple system has been successfully separated from Quotations with full support for:
- ✅ Module variables from Company, Quotation, Inventory, Employees, and more
- ✅ Dynamic variable selection in the UI (Module → Variable)
- ✅ Material table column configuration with variable binding
- ✅ Type-safe variables with proper filtering

---

## 📱 Using the Temple Editor UI

### Access the Temple Editor
```
Frontend Route: /admin/temples
```

### Create a New Template

1. **Navigate to Templates Page**
   - Go to `/admin/temples`
   - Click "Create New Template"

2. **Fill in Basic Information**
   - Enter template name (required)
   - Add optional description

3. **Configure Material Table Columns**
   - Go to "Material Table" tab
   - Click "+ Add Column"
   - Fill in column details:
     - **Column Label**: Display name (e.g., "Material Code")
     - **Width**: Column width in pixels (30-500)
     - **Text Alignment**: Left, Center, or Right

4. **Bind Column to Variable**
   - In the column editor dialog
   - Select **Module** dropdown
   - Select **Variable** from that module
   - Only variables with compatible types are shown:
     - ✅ string
     - ✅ number
     - ✅ date
     - ✅ boolean
     - ❌ image (not for table columns)

5. **Review & Save**
   - Go to "Preview" tab to see layout
   - Click "Create Template" to save

---

## 🔧 Backend API Usage

### Fetch Available Variables

**Endpoint**: `GET /api/temples/variables`

**Query Parameters**:
- `modules` (optional): Comma-separated list (e.g., `inventory,employees`)

**Example**:
```bash
# Get all variables
curl http://localhost:4000/api/temples/variables

# Get specific modules
curl "http://localhost:4000/api/temples/variables?modules=inventory,employees"

# Get compatible (non-image) variables for table columns
curl http://localhost:4000/api/temples/variables/compatible
```

**Response**:
```json
{
  "status": "ok",
  "data": {
    "company": [
      {
        "key": "company.logo",
        "label": "Company Logo",
        "type": "image",
        "module": "quotations",
        "description": "Company logo image",
        "sourceFields": ["logo"]
      },
      {
        "key": "company.name",
        "label": "Company Name",
        "type": "string",
        "module": "quotations",
        "description": "Official company name",
        "sourceFields": ["companyName"]
      }
    ],
    "inventory": [
      {
        "key": "material.code",
        "label": "Material Code",
        "type": "string",
        "module": "inventory"
      }
    ]
  }
}
```

---

## 🧩 Integrating into React Components

### Using the Temple Editor Component

```tsx
import { TempleEditor } from '@/features/temples/temple-editor';
import { Temple } from '@/types/temples';

export function MyComponent() {
  const handleSave = (temple: Temple) => {
    console.log('Temple saved:', temple);
  };

  return (
    <TempleEditor
      templeId={optionalIdForEditing}
      onSave={handleSave}
      onCancel={() => console.log('Cancelled')}
    />
  );
}
```

### Using the Module Variable Selector

```tsx
import { ModuleVariableSelector } from '@/features/temples/module-variable-selector';

export function MyComponent() {
  const handleSelect = (variableKey: string) => {
    console.log('Selected variable:', variableKey);
  };

  return (
    <ModuleVariableSelector
      onSelect={handleSelect}
      allowedTypes={['string', 'number']}  // Optional: filter by type
      excludeModule="quotation"             // Optional: exclude specific module
    />
  );
}
```

### Using the Material Table Column Editor

```tsx
import { MaterialTableColumnEditor } from '@/features/temples/material-table-column-editor';
import { useTempleVariables } from '@/hooks/useTempleVariables';

export function MyComponent() {
  const { variables } = useTempleVariables();
  const [columns, setColumns] = useState([]);

  return (
    <MaterialTableColumnEditor
      columns={columns}
      onColumnsChange={setColumns}
      availableVariables={variables || {}}
    />
  );
}
```

### Using the Hook to Fetch Variables

```tsx
import { useTempleVariables } from '@/hooks/useTempleVariables';

export function MyComponent() {
  const {
    variables,      // Record<string, VariableDefinition[]>
    isLoading,      // boolean
    error,          // Error | null
    refetch         // () => Promise<void>
  } = useTempleVariables();

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  // Use variables...
  Object.entries(variables).forEach(([module, vars]) => {
    console.log(`${module}:`, vars);
  });
}
```

---

## 📊 Available Variables Reference

### Company Module (10 variables)
```
company.logo          - image   - Company logo image
company.seal          - image   - Company seal/stamp
company.name          - string  - Official company name
company.taxCode       - string  - Tax registration code
company.address       - string  - Business address
company.phone         - string  - Phone number
company.email         - string  - Email address
company.bankAccount   - string  - Bank account number
company.bankName      - string  - Bank name
company.bankBranch    - string  - Bank branch
```

### Quotation Module (8 variables)
```
quotation.code        - string  - Quotation reference number
quotation.date        - date    - Quotation date
quotation.type        - string  - Type (commercial, technical, etc.)
project.name          - string  - Associated project
customer.name         - string  - Customer name
customer.recipient    - string  - Recipient name
customer.request      - string  - Special requests
quotation.content     - string  - Additional notes
```

### Money Module (6 variables)
```
money.subtotalOneSet  - number  - Subtotal for one set
money.numberOfSets    - number  - Quantity of sets
money.totalBeforeVat  - number  - Total before VAT
money.vatRate         - number  - VAT percentage
money.vatAmount       - number  - VAT amount
money.grandTotal      - number  - Final total
```

### Inventory Module (12 variables)
```
material.code         - string  - Material code/SKU
material.name         - string  - Material name
material.model        - string  - Model/variant
material.brand        - string  - Brand name
material.origin       - string  - Country of origin
material.unit         - string  - Unit of measure
material.price        - number  - Unit price
material.quantity     - number  - Stock quantity
material.leadTime     - number  - Delivery time (days)
material.warranty     - number  - Warranty (months)
material.picture      - image   - Product image
material.remark       - string  - Notes
```

### Employees Module (5 variables)
```
employee.name         - string  - Full name
employee.email        - string  - Work email
employee.phone        - string  - Work phone
employee.department   - string  - Department
employee.position     - string  - Job title
```

### Signature Module (3 variables)
```
signature.image       - image   - Signature image
signature.name        - string  - Signer name
signature.title       - string  - Signer title
```

---

## 🔌 Connecting to Quotations

### Link Quotation to Temple

```tsx
// Create quotation with temple reference
const quotation = await createQuotation({
  // ... other fields
  templeId: selectedTempleId,  // NEW: reference to temple
});
```

### In Quotation Detail, Use Temple Variables

```tsx
// Resolve variable value from template binding
const variableValue = await resolveVariableValue(
  'inventory.material.code',  // variable key
  { quotation, company, item }  // data context
);
```

### Material Table Uses Temple Columns

```tsx
// Fetch temple configuration
const temple = await fetchTemple(quotationTempleId);
const tableConfig = temple.versions[0]?.tableConfig;

// Render columns based on template
{tableConfig?.columns.map(column => (
  <td key={column.key}>
    {/* Resolve binding: column.variableKey → actual data */}
  </td>
))}
```

---

## 🚀 API Reference

### Temple CRUD

**Create Temple**
```bash
POST /api/temples
Content-Type: application/json

{
  "name": "Standard Template",
  "description": "Main quotation template",
  "tableConfig": {
    "columns": [
      {
        "key": "col-1",
        "label": "Material Code",
        "variableKey": "inventory.material.code",
        "width": 120,
        "visible": true,
        "align": "left"
      }
    ]
  }
}
```

**Get Temple**
```bash
GET /api/temples/{id}
```

**List Temples**
```bash
GET /api/temples?skip=0&take=20&search=keyword&status=active
```

**Update Temple**
```bash
PUT /api/temples/{id}
Content-Type: application/json

{
  "name": "Updated Name",
  "description": "Updated description"
}
```

**Delete Temple**
```bash
DELETE /api/temples/{id}
```

### Temple Versions

**Get Latest Version**
```bash
GET /api/temples/{id}/latest-version
```

**Create New Version**
```bash
POST /api/temples/{id}/versions
Content-Type: application/json

{
  "tableConfig": { ... }
}
```

### Variable Bindings

**Get Bindings**
```bash
GET /api/temples/{id}/variable-bindings
```

**Create Binding**
```bash
POST /api/temples/{id}/variable-bindings
Content-Type: application/json

{
  "columnKey": "col-1",
  "variableKey": "inventory.material.code"
}
```

---

## 📚 Helper Functions

### Frontend Utilities

```tsx
import {
  filterVariablesByType,
  getListTypeVariables,
  getTableColumnVariables,
  getModulesFromVariables
} from '@/hooks/useTempleVariables';

// Filter by specific type
const stringVars = filterVariablesByType(variables, 'string');

// Get only list-type variables
const listVars = getListTypeVariables(variables);

// Get table-compatible variables (exclude images)
const tableVars = getTableColumnVariables(variables);

// Get available modules
const modules = getModulesFromVariables(variables);
```

### Backend Utilities

```typescript
import {
  getVariableByKey,
  getVariablesByModule,
  getAllVariables,
  getTableColumnVariables
} from '@/modules/temples/temples.variables';

// Get specific variable
const variable = getVariableByKey('material.code');

// Get module variables
const inventoryVars = getVariablesByModule('inventory');

// Get all variables
const allVars = getAllVariables();

// Get table-compatible
const tableVars = getTableColumnVariables();
```

---

## 🧪 Testing Examples

### Test 1: Fetch Variables
```bash
curl http://localhost:4000/api/temples/variables | head -100
# Expected: Variables grouped by module with proper metadata
```

### Test 2: Create Temple
```bash
curl -X POST http://localhost:4000/api/temples \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Template",
    "description": "Test",
    "tableConfig": {
      "columns": [{
        "key": "col-1",
        "label": "Code",
        "variableKey": "inventory.material.code",
        "width": 120,
        "visible": true,
        "align": "left"
      }]
    }
  }'
```

### Test 3: Get Temple
```bash
curl http://localhost:4000/api/temples/{temple-id}
```

### Test 4: Create Variable Binding
```bash
curl -X POST http://localhost:4000/api/temples/{temple-id}/variable-bindings \
  -H "Content-Type: application/json" \
  -d '{
    "columnKey": "col-1",
    "variableKey": "inventory.material.name"
  }'
```

---

## 📋 Troubleshooting

### Variables Not Showing in UI
1. Check browser console for fetch errors
2. Verify API endpoint: `GET /api/temples/variables`
3. Ensure API server is running on port 4000

### Variable Binding Not Working
1. Verify variable key format: `module.property.path`
2. Check variable type is compatible (exclude `image` for table columns)
3. Verify temple was saved with table config

### Material Table Not Rendering
1. Check if temple has `tableConfig` in version
2. Verify column `variableKey` is set
3. Ensure quotation has data for the variable

---

## 🎓 Next Steps

1. **Implement in Quotation Editor**
   - Reference template by ID
   - Use template columns for material table
   - Resolve variables to actual data

2. **Create Template Manager Page**
   - List all templates
   - Edit/delete templates
   - Version control UI

3. **Add Layout Editor**
   - Drag-drop blocks
   - Configure company info block
   - Configure totals block

---

## 📞 Support

For issues or questions:
1. Check the implementation summary: `TEMPLE_IMPLEMENTATION_SUMMARY.md`
2. Review API responses for error messages
3. Check browser console and API server logs
4. Contact development team

---

**Last Updated**: 2026-05-22  
**Status**: Ready for Production  
**Version**: 1.0
