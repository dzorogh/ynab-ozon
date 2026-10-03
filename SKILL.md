---
name: ynab-ozon
description: >-
  Match Ozon orders to bank transactions in a shared family YNAB budget, fix
  wrong payees and fake transfers, set categories, memos and splits, then
  approve. Supports several people, each with their own Ozon account and card,
  via profiles in ids.md. Use when the user asks to раскидать Озон, разнести
  покупки Озона, выгрузил операции, поправить оранжевый флаг, or when imported
  payees (own name like «Иван И.», «Себе В Другой Банк», a random shop) are
  actually Ozon purchases.
---

# YNAB ↔ Ozon

Reply in the user's language. Never print the YNAB token or Ozon PII (phone, email, address, DOB, Ozon user id).

Budget, accounts, people and categories live in [ids.md](ids.md). If it is missing, copy [ids.example.md](ids.example.md) to `ids.md` and fill it in with the user (`GET /budgets`, `/accounts`, `/categories`, `/payees`).

Token: `YNAB_TOKEN` from a local gitignored `.env`. If it is missing, create the file with an empty key, open it for the user, and wait. Never ask for the token in chat.

## 1. Pick the profile

Each profile in `ids.md` has a name, a spending account, the person's own Ozon account, and the self-name the bank uses in imports.

- If the request names the person («мои», «жены», by name), use that profile.
- Unclear → check every profile's spending account for orange / unapproved rows. Do one profile at a time; each needs its own Ozon login.
- Before matching, confirm the Ozon session belongs to that person: read `window.__NUXT__.state` → `userInfo.user.firstName` in the browser and compare it to the profile without printing other fields. Wrong person → unlock the browser and ask them to log in.
- Order ids look like `<ozonUserId>-NNNN`. Take the prefix from the order links on the list page each run. Never store it or reuse another person's.

## 2. Find the rows to fix

`GET /budgets/{budget}/accounts/{account}/transactions?since_date=…` on the profile's spending account. Candidates:

| What you see | Reality |
|---|---|
| Orange flag | User already marked it as wrong |
| Own self-name or a random payee, category Inflow, purchase-sized outflow | Ozon SBP payment |
| Transfer → own Ozon Bank savings, import like «Себе В Другой Банк», purchase-sized | Ozon purchase, not a savings move |
| Ozon payee, unapproved or Uncategorized | Ozon purchase that needs a category |
| Inflow with own self-name matching a cancelled order | Ozon refund |

Do not touch:

- Large real savings moves and deposits (round sums, far above any order).
- Transfers between family members. Another member's self-name on this account is usually a real transfer, not Ozon.
- Anything listed under the profile's «do not touch» in `ids.md`, and clearly non-Ozon payees.
- A small fixed monthly `Ozon` charge is the Ozon Premium subscription → Подписки, not an order.

## 3. Read Ozon orders

1. Use the **cursor-ide-browser**: list tabs, then lock an existing Ozon tab or navigate to `https://www.ozon.ru/my/orderlist`. Antibot or not logged in → unlock and ask the profile's owner to sign in. If Browser MCP drops after login, fall back to a logged-in Chrome via AppleScript `execute javascript` (store async results in `window.__x`, read them after a pause).
2. Collect `a[href*="orderdetails"]` from the «Актуальные» and «Завершённые» tabs.
3. Fetch each `https://www.ozon.ru/my/orderdetails/?order=<id>` with `credentials: 'include'`. Parse the date, status, paid total («Оплачено … ₽»), item names and prices. Start from the profile's last order in `ids.md` and walk forward until pages come back empty.
4. Collapsed carts: click «Показать ещё N товара» via `button.click()` inside `Runtime.evaluate` when the accessibility click does nothing.

## 4. Match

- Exact amount + nearby date = high confidence.
- Two charges can fund two orders as a pair (`A+B` charges = `C+D` orders).
- Charge **less** than the order: the rest came from the Ozon Bank balance (cancelled-order refunds). Example: a 923 ₽ order with a 617 ₽ charge means 306 ₽ came from the bank balance.
- Refund (`+`): same category as the cancelled spend. A partial refund often means a replacement was paid out of it (`15546 − 3139 = 12407`).
- A cancelled line refunded to Ozon Bank that then paid another order: book that amount to the **kept** item's category.
- An order with no card charge was paid entirely from the Ozon balance. Mention it; do not invent a transaction.

## 5. Categories

Map goods to the budget's own categories listed in `ids.md`. Default mapping for a typical family budget:

| Goods | Category |
|---|---|
| Food, Ozon fresh, coffee | Продукты |
| Diapers, baby food, kids' clothes/toys, stroller parts, child gear | Ребенок |
| Adult clothes, shoes | Одежда |
| Cases, chargers, cables, surge protectors, gadgets | Электроника и девайсы |
| Kitchenware, storage, household consumables (salt, bleach, bags, cleaners) | Мебель и товары для дома |
| Tools, hoses, LED drivers, film, shower heads, fixtures | Обслуживание и ремонт дома |
| Cosmetics, shaving, pharmacy, repellent | Красота и медицина |
| Pet food, toys, collars, beds | Животные |
| Presents for others | Подарки и помощь |
| Fishing tackle, hobby | Развлечения |
| Ozon Premium | Подписки |

Дом or ремонт: fixtures and tools → ремонт; stored goods and consumables → дом. Ребенок wins when the item is clearly for the child. Ask only when the item is genuinely ambiguous; apply user corrections immediately (e.g. bearings for a stroller → Ребенок).

## 6. Write

`PATCH /v1/budgets/{budget}/transactions`. Milliunits: `1 ₽ = 1000`, outflow negative.

```json
{
  "id": "<txn>",
  "account_id": "<profile spending account>",
  "payee_id": "<Ozon payee>",
  "category_id": "<cat, or null when split>",
  "memo": "NNNN: short item list; bank math if any",
  "approved": true,
  "flag_color": null,
  "subtransactions": [
    {"amount": -409000, "payee_id": "<ozon payee>", "category_id": "<cat>", "memo": "шланг 2 м"}
  ]
}
```

- Omit `subtransactions` for a single category. Splits must sum to the parent.
- Converting a fake transfer: set `payee_id` to Ozon. YNAB deletes the paired inflow on the savings account.
- Memo uses the short order number (`0780`) plus the items, never the Ozon user id.
- Apply as soon as the match is solid.

## 7. Verify and report

- The fixed rows: Ozon payee, a category, approved, no flag.
- Savings accounts have no leftover purchase-sized inflows; real transfers remain.
- Update the profile's last order in `ids.md`.
- Reply with a table (date, amount, order, items, category) and explain the bank-balance math and cancelled lines.

## Do not

- Mix profiles: never match one person's YNAB rows against another person's Ozon orders.
- Leave test rows in YNAB.
- Commit `ids.md`, tokens, Ozon user ids or personal amounts.
