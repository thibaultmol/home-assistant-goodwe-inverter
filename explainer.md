# Fix for Issue #409: Migration Handler Missing Error

## Problem

Users switching from the **official Home Assistant Goodwe integration** to the **HACS version** encountered this error:

```
Migration handler not found for entry GoodWe INV1 for goodwe
```

The integration would fail to load after the switch.

## Root Cause

This is a **config entry version mismatch**:

| Integration | Config Entry Version |
|------------|---------------------|
| Official HA Goodwe | Version **3** |
| HACS Goodwe | Version **2** |

When users had the official integration configured (version 3) and then installed the HACS version (which only supported up to version 2), Home Assistant tried to migrate the config entry but found no migration handler for going from version 3 → version 2.

The HACS integration's `async_migrate_entry()` function had this check:

```python
if config_entry.version > 2:
    # This means the user has downgraded from a future version
    return False  # Migration fails!
```

This rejected any config entry with version > 2, causing the error.

## The Fix

Two changes were required:

### 1. Update `config_flow.py` - Bump VERSION to 3

```python
class GoodweFlowHandler(ConfigFlow, domain=DOMAIN):
    """Handle a Goodwe config flow."""

    VERSION = 3  # Added this line to match official integration
    MINOR_VERSION = 2
```

This ensures new installations create version 3 config entries, matching the official integration.

### 2. Update `__init__.py` - Add migration handler for version 2 → 3

The `async_migrate_entry()` function was updated to:

1. Accept version 3 as valid (no longer reject it)
2. Add a migration path from version 2 → version 3

```python
async def async_migrate_entry(
    hass: HomeAssistant, config_entry: GoodweConfigEntry
) -> bool:
    """Migrate old config entries."""

    version = config_entry.version
    data = dict(config_entry.data)

    if version > 3:
        _LOGGER.error(
            "Config entry %s is version %s, newer than supported version %s",
            config_entry.title,
            version,
            3,
        )
        return False

    # ... version 1 → 2 migration (unchanged) ...

    if version == 2:
        # Migrate from version 2 to version 3
        host = data[CONF_HOST]
        port = data.get(CONF_PORT)
        if not port:
            try:
                _, port = await GoodweFlowHandler.async_detect_inverter_port(host=host)
            except InverterError as err:
                raise ConfigEntryNotReady from err
        protocol = data.get(CONF_PROTOCOL)
        if not protocol:
            protocol = "TCP" if port == GOODWE_TCP_PORT else "UDP"
        new_data = {
            CONF_HOST: host,
            CONF_PORT: port,
            CONF_PROTOCOL: protocol,
            # ... other fields ...
        }
        hass.config_entries.async_update_entry(config_entry, data=new_data, version=3)
        data = new_data

    return True
```

## Why This Works

1. **Version 3 config entries** (from official integration) are now accepted as valid - no migration needed since the data structure is compatible
2. **Version 2 config entries** (older HACS installs) get migrated to version 3
3. **Version 1 config entries** still get migrated 1 → 2 → 3

This allows seamless switching between the official and HACS versions of the integration.

## Reference

- Original issue: https://github.com/mletenay/home-assistant-goodwe-inverter/issues/409
- Home Assistant Config Entry documentation: https://developers.home-assistant.io/docs/config_entries_index