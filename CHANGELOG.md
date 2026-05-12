7 August 2025 - `Feature` Enhanced End-User Security in TonConnect:
  - Added `/verify` endpoint for connection verification with statuses: `ok`, `danger`, `warning`, `unknown`
  - Introduced encrypted `request_source` metadata in bridge messages containing origin, IP, User-Agent, timestamp, and client ID
  - Added BridgeRequestSource struct for request source tracking and verification
  - Updated bridge message format to include optional `request_source` field
  - Updated bridge message format to include `connect_source` field
  - Enhanced wallet workflows with connection and transaction verification processes
  - Added comprehensive wallet implementation guidelines for security features
  - Implemented phased rollout approach for verification status responses
  - Maintained full backward compatibility with existing dApps and bridges
  - Added user interface guidelines for verification marks and security warnings

7 March 2023 - `Feature` format of `DeviceInfo` of `ConnectEventSuccess` changed; Parameter `maxMessages` added to the Feature `SendTransaction`.