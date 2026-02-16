# Employee Advance (Vue.js Frontend)

Vue 3 + Vite frontend for the Employee Advance screen with **Advance Payment** and **Advance Adjustment** modes.

## Run locally

```bash
npm install
npm run dev
```

Open the URL shown in the terminal (e.g. http://localhost:5173).

## Build

```bash
npm run build
```

Output is in `dist/`.

## Features

- **Advance Payment**: Advance Purpose, EIN, Employee Name, Advance Date (ERP Open), Payment Method, Advance Amount, Previous Advance Balance, Reference PR/Demand (GetData), Remarks, and action buttons (Reverse, Edit, Save).
- **Advance Adjustment**: Same header fields with Advance Adjustment Date, detail table (Adv. Date, Purpose, Amt., Bill Amt. For Adj., Cash Receive/(Pay), Remaining Balance, Particulars, Reference), totals row, summary (Total Bill Amount, Total Cash Amount, Total Adjusted Adv. Amt.), Attach Bill Document with doc list, and action buttons.

Switch between modes using the radio buttons at the top.
