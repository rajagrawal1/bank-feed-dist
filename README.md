# bank-feed-dist

A signed file of public facts about Indian banks, from RBI and DICGC.

- `banks.signed.json`: one JSON envelope `{"format":1,"keyId","sig","payload"}`. `sig` is an Ed25519 signature
  over the UTF-8 bytes of `payload`, which is the feed itself as a JSON string.
- `ed25519_pub.pem`: the public key that signs it.

What is in it: for about 1,900 Indian banks, whether the Reserve Bank of India has announced restrictions on the
bank, its deposit insurance status on DICGC's list, whether it merged or lost its licence, and its RBI record for
the last five years, each item dated and linked to the RBI release. Only RBI press releases and DICGC's published
lists are used. It states what was announced; it never rates or judges a bank.

Files are replaced by an automated build; please do not open pull requests.
RBI and DICGC content is public record, reproduced with attribution.
