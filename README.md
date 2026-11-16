# protocol-proxy-mqtt

MQTT plugin for [protocol-proxy](https://github.com/eclipse-volttron/lib-protocol-proxy). A
`ProtocolProxyManager` launches one `MQTTProxy` subprocess per broker connection:

```python
from protocol_proxy.manager.gevent import GeventProtocolProxyManager
manager = GeventProtocolProxyManager.get_manager('mqtt')   # resolves protocol_proxy.protocol.mqtt.PROXY_CLASS
peer = manager.get_proxy(('mqtt', 'broker.example.org', 1883), host='broker.example.org', port=1883)
manager.wait_peer_registered(peer, timeout=30)
```

Keyword arguments to `get_proxy` become command line options of the proxy process:
`host`, `port`, `keepalive`, `bind_address`, `bind_port`, `client_id`, `username`, `password`,
`tls`, `protocol` (`MQTTv31`, `MQTTv311`, `MQTTv5`), `qos`, `reconnect_min_delay`,
`reconnect_max_delay`.

## Messages

| Direction | Method | Payload (JSON) |
| --- | --- | --- |
| broker → manager | `PUBLISH_LOCAL` | `{"topic", "payload": <hex bytes>, "qos", "retain", "mid"}` |
| manager → broker | `PUBLISH_REMOTE` | `{"topic", "payload", "qos"?, "retain"?}` |
| manager → broker | `SUBSCRIBE_REMOTE` | `{"topics": [filter \| [filter, qos] \| {"topic", "qos"}]}` |
| manager → broker | `UNSUBSCRIBE_REMOTE` | `{"topics": [...]}` |

String payloads are sent as UTF-8; other JSON values are serialized with `json.dumps`.
Subscriptions are remembered and restored after every reconnection. The proxy reconnects to the
broker with exponential backoff (`reconnect_min_delay` 1 s to `reconnect_max_delay` 10 s by
default) and registers with the manager independently of broker availability.

While the broker is unreachable, `PUBLISH_REMOTE` requests at QoS 0 are dropped and requests at
QoS 1 or 2 are held by the client and sent on reconnection. The log carries one warning when the
broker is lost (and one for the first dropped publish), the retries at debug level, and one line
on reconnection with how long the broker was gone and how many publishes were dropped or held.

## Development

```bash
pip install -e .
pytest
```

The integration test launches a real proxy subprocess against a manager; it needs no broker.
