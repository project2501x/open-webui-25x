# Sample Inline Suggestion Template

## Template Content
```
Summarise the most recent earnings transcript for {{inline-suggestion}}
```

## Usage Instructions
1. Copy this template content
2. Create a new prompt in the application
3. Paste the template content
4ve the prompt with a command like `/transcript`

## Expected Behavior
When a user types `/transcript` and selects this template:
1mplate is inserted into the chat input
2{inline-suggestion}}` placeholder is replaced with an inline input field
3he input field has a placeholderEnter suggestion...
4. User can type in the field and see autocomplete suggestions
5. Suggestions include company names likeApple Inc. (AAPL)", "Microsoft Corporation (MSFT)", etc.
6 User can select a suggestion with Tab/Enter or by clicking
7. The selected value replaces the input field

## Example Output
After selectingApple Inc. (AAPL)", the final prompt would be:
```
Summarise the most recent earnings transcript for Apple Inc. (AAPL)
```

## Available Suggestions
- Apple Inc. (AAPL)
- Microsoft Corporation (MSFT)
- Amazon.com Inc. (AMZN)
- Google LLC (GOOGL)
- Tesla Inc. (TSLA)
- Meta Platforms Inc. (META)
- Netflix Inc. (NFLX)
- Salesforce Inc. (CRM)
- Adobe Inc. (ADBE)
- Intel Corporation (INTC) 