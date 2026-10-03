# YNAB IDs (template)

Copy to `ids.md` (gitignored) and fill in. Get IDs from the YNAB API: `GET /budgets`, `/budgets/{id}/accounts`, `/categories`, `/payees`.

Budget: `<budget-id>`

Ozon payee (`Озон Банк (Ozon)` or similar): `<payee-id>`

## Profiles

### <Person A>

- Spending account (card Ozon is paid from): `<account-id>`
- Self-name in bank imports: `<Имя Ф.>`
- Ozon Bank savings where fake transfers land: `<account-id>` (optional)
- Do not touch: `<real deposits, loans, salary payees>`
- Last order applied: `—`

### <Person B>

- Spending account: `<account-id>`
- Self-name in bank imports: `<Имя Ф.>`
- Do not touch: `<…>`
- Last order applied: `—`

## Categories

| Name | ID |
|---|---|
| Продукты | `<id>` |
| Ребенок | `<id>` |
| Одежда | `<id>` |
| Электроника и девайсы | `<id>` |
| Мебель и товары для дома | `<id>` |
| Обслуживание и ремонт дома | `<id>` |
| Красота и медицина | `<id>` |
| Животные | `<id>` |
| Подарки и помощь | `<id>` |
| Развлечения | `<id>` |
| Подписки | `<id>` |

If a PATCH returns 400 on `category_id`, refresh with `GET /budgets/{budget}/categories`.
