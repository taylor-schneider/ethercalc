# Example Usage

## UDF Example
```javascript
// Define a user-defined function (UDF)
SocialCalc.Formula.RegisterUserFunction("DOUBLE", 1, function(value) {
  return value.value * 2;
});

// Use in a cell: =DOUBLE(A1)
```

## Macro Example
```javascript
// Define a macro
SocialCalc.Macros.register("FormatReport", function(params) {
  this.setText("A1", "Sales Report");
  this.formatCell("A1", { font: "bold", bgcolor: "#CCCCFF" });
  this.setFormula("A2", "SUM(A3:A10)");
});

// Execute: SocialCalc.Macros.execute("FormatReport", sheet, {})

// Note: SocialCalc.Formula and RegisterUserFunction are existing SocialCalc APIs.

Excel analogies:
- UDF Example = custom function used in a formula.
- Macro Example = VBA macro that edits cells and formatting.

```
