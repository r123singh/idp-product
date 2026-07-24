# Document Schemas

Extraction is **schema-driven**. Each document type declares:
- Fields (keys, labels, types, patterns)
- Validation rules (including natural-language / pattern checks)
- Optional summary guidance

## Example types
| Type | Purpose |
|------|---------|
| `shareTransferCertificate` | Primary transfer instrument |
| `passport` | Party identity + expiry |
| `emiratesId` / resident ID | Local identity support |
| `proofOfAddress` | Address evidence |

## Entity numbers
Examples use a generic `ENT` + digits pattern (client-specific prefixes removed for publication).

Schemas ship in `idp-product/schemas` and alongside the extraction engine.
