---
name: human-like-code-generator
description: >
  Generates code that mimics experienced human developer practices, prioritizing readability,
  maintainability, and intentional structure over cleverness or concision. Use this skill
  whenever writing code in Golang or Java/Spring MVC, or when the user asks for code
  generation, implementation help, or refactoring. Also trigger when the user asks for
  Thymeleaf fragments, REST controllers, service layers, DTOs, or any backend/frontend
  code in these stacks. If any code is being written, this skill applies.
---

# Human-Like Code Generator

Generate code the way an experienced human developer would write it: clear, explicit, and
maintainable. The goal is code that reads like prose and communicates intent without relying
on comments to explain dense constructs.

---

## Language Stack

Primary languages are **Golang** and **Java with Spring MVC**. Frontend uses **HTML5 + Thymeleaf**.
Default to these unless the user specifies otherwise.

---

## Core Philosophy

- Clarity over cleverness. If a construct requires mental unwrapping, rewrite it.
- Explicit over implicit. Name things what they are.
- Flat over nested. Reduce indentation depth wherever possible.
- Small functions over large ones. Each function does one thing and is named for it.

---

## Naming Rules

**Variables**: Use full descriptive words that reflect domain meaning.
- Good: `totalOrderPrice`, `activeUserCount`, `pendingInvoiceList`
- Bad: `tot`, `cnt`, `data`, `res`, `tmp`

**Functions/Methods**: Use intention-revealing verbs.
- Good: `calculateTotalOrderPrice`, `fetchUserById`, `buildInvoiceSummary`
- Bad: `processData`, `handleStuff`, `doThing`

Single-letter variables are only acceptable in trivial `for` loop indices (`i`, `j`) and
nothing else.

---

## Java / Spring MVC Rules

### Lombok
Use Lombok to eliminate boilerplate on data classes and DTOs. Preferred annotations:
- `@Getter`, `@Setter`
- `@Builder`
- `@AllArgsConstructor`, `@NoArgsConstructor`
- `@Data` for simple value objects

Avoid Lombok on classes where the generated behaviour would obscure logic.

### Lambdas and Streams
Avoid lambdas and streams unless the user explicitly requests them. Use named loops and
explicit iteration instead. The goal is flow that reads top to bottom without needing to
parse a pipeline.

Bad:
```java
return items.stream()
    .map(i -> i.getPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
    .reduce(BigDecimal.ZERO, BigDecimal::add);
```

Good:
```java
public BigDecimal calculateTotalOrderPrice(List<OrderItem> orderItems) {
    BigDecimal totalPrice = BigDecimal.ZERO;

    for (OrderItem item : orderItems) {
        BigDecimal itemTotal = item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity()));
        totalPrice = totalPrice.add(itemTotal);
    }

    return totalPrice;
}
```

### Error Handling
- Use meaningful exception messages that describe what went wrong and why.
- Never swallow exceptions silently.
- Prefer specific exception types over generic `RuntimeException` where it adds clarity.

### Structure
- Controller methods delegate to service methods. No business logic in controllers.
- Service methods are named for their action, not their HTTP verb.
- Repository interfaces follow Spring Data naming conventions.

---

## Golang Rules

- Use named return types sparingly; prefer explicit returns.
- Error handling is always explicit. Check every error. Name error variables `err`.
- Struct fields are descriptive. No single-letter field names.
- Keep functions short. If a function is getting long, extract named helper functions.
- Use early returns to avoid deep nesting.

Good:
```go
func calculateDiscountedPrice(originalPrice float64, discountPercent float64) (float64, error) {
    if discountPercent < 0 || discountPercent > 100 {
        return 0, fmt.Errorf("discount percent must be between 0 and 100, got %f", discountPercent)
    }

    discountAmount := originalPrice * (discountPercent / 100)
    discountedPrice := originalPrice - discountAmount

    return discountedPrice, nil
}
```

---

## Thymeleaf / HTML Rules

When producing Thymeleaf fragments or templates, always split output into three separate files:

1. `fragment-name.html` - markup only, Thymeleaf attributes, no inline `<style>` or `<script>`
2. `fragment-name.css` - all styles for the fragment
3. `fragment-name.js` - all JavaScript for the fragment

Reference the CSS and JS from the HTML using standard `<link>` and `<script>` tags, or note
clearly where they should be included in the layout template.

Keep Thymeleaf expressions readable. Avoid chaining multiple operations inline in `th:`
attributes. If the logic is complex, move it to the model in the controller.

---

## Comments

- Explain why, not what. Code should be self-explanatory enough that "what" is obvious.
- Add a comment when a decision might look wrong to a future reader but is intentional.
- No redundant comments that restate what the variable name or method name already says.
- Skip comments entirely when the code is self-evident.

---

## Formatting

- Blank lines between logical blocks within a method.
- Group related fields together in class definitions.
- Keep line length reasonable (around 120 characters max for Java, 100 for Go).
- Consistent indentation throughout.

---

## Output Behaviour

When generating code:
1. Think through the structure before writing.
2. For complex tasks, briefly outline the approach before the code.
3. Write incrementally - controller, then service, then repository, not all in one wall.
4. Do not over-optimise prematurely.
5. Do not compress logic for brevity at the cost of clarity.
