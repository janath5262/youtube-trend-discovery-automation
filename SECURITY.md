# Security and publishing checklist

## Immediate action: rotate the YouTube API key

The **original** uploaded n8n export contained a plaintext YouTube API key in the `Get Video Data` HTTP Request node. This sanitized repository deliberately removes it.

1. In Google Cloud Console, locate the key used by the original workflow.
2. Revoke/delete the exposed key and create a new restricted key. Configure appropriate API restrictions for YouTube Data API v3 and applicable application restrictions.
3. Store the replacement in an n8n **Query Auth credential** named `key`; do not paste it into a public workflow JSON.
4. Test the configured HTTP Request step after rotation.

## Public repository checks

- [ ] Search all files and Git history for `AIza`, API keys, OAuth tokens, passwords, bearer tokens, and session cookies.
- [ ] Do not publish the original export (it contains a key).
- [ ] Do not upload `.env`, credentials exports, n8n backup databases, or actual Airtable customer records.
- [ ] Use example/placeholder Airtable base and table IDs; configure your own IDs on import.
- [ ] Remove private webhook URLs, n8n instance IDs, and credential references from public artifacts.
- [ ] Blur screenshots to hide private credentials, API responses, account details, or customer information.
- [ ] Confirm the workflow remains unpublished/inactive until credentials are configured securely.

No actual API key or OAuth credential is included in this public workflow template.
