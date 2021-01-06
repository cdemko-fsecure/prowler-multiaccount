This file documents a few examples of how to configure the Lambda function and Prowler scans. Additional examples will be added as new features are added.

These JSON objects are the contents of the EventBridge event that target the Initiator Lambda function (using the `Constant (JSON text)` input type). The frequency of these events is up to the user to configure.

### Scan all accounts in Organization with a full scan (every Prowler check)

    {}

### Scan all accounts in Organization with only CIS Level 1 checks

    {"group": "cislevel1"}

*Pretty print*

```
{
  "group": "cislevel1"
}
```

### Scan all accounts in a single OU with only CIS level 1 checks

    {"group": "cislevel1", "ou": ["ou-abc2-a1b2c3d4"]}

*Pretty print*

```
{
  "group": "cislevel1",
  "ou": [
    "ou-abc2-a1b2c3d4"
  ]
}
```

### Scan all accounts in multiple OUs (with a root) with only Extra checks

    {"group": "extras", "ou": ["ou-abc2-a1b2c3d4", "ou-abc2-a4b3c2d1", "r-abc2"]}

*Pretty print*

```
{
  "group": "extras",
  "ou": [
    "ou-abc2-a1b2c3d4",
    "ou-abc2-a4b3c2d1",
    "r-abc2"
  ]
}
```
