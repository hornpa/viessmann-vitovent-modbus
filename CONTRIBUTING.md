# Contributing

Contributions that confirm, correct, or extend the reverse-engineered register map are welcome.

## Before submitting

- State the exact Vitovent model/variant and, if known, firmware revision.
- Describe the test setup and Modbus interface used.
- Distinguish observed behaviour from assumptions or hypotheses.
- Do not probe undocumented writable registers destructively.
- Do not include credentials, private network addresses, serial numbers, or other sensitive data.

## Register-map changes

For a new or corrected register, include the address, function code, access type, observed values/scaling, test method, and confidence level. Raw scan or capture data is particularly useful when it can be shared safely.

## Pull requests

Keep changes focused and explain how they were verified. Documentation and data contributions are accepted under CC BY 4.0; code snippets and Loxone templates are accepted under MIT, as described in `LICENSE`.
