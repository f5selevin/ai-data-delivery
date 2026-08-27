# Documentation Source

The documentation in this folder is based specifically on **Class 13** from the [F5 Agility Labs ADC repository](https://github.com/f5devcentral/f5-agility-labs-adc.git).

Source: [Class 13 documentation](https://github.com/f5devcentral/f5-agility-labs-adc/tree/develop/docs/class13)

## Documentation Viewer

The [Partner Spec Guide Viewer](https://github.com/f5selevin/partner-spec-guide-view) can be used to view this documentation locally.

## Install the Lab Guide Server on UDF

Run the following command on the UDF host to register and install the lab guide server:

```shell
INSTALL_DIR="$(mktemp -d)" && \
curl -fsSL https://raw.githubusercontent.com/f5selevin/ai-data-delivery/main/udf/labguide/register-labguide-server.sh -o "${INSTALL_DIR}/register-labguide-server.sh" && \
curl -fsSL https://raw.githubusercontent.com/f5selevin/ai-data-delivery/main/udf/labguide/create-labguide-server.sh -o "${INSTALL_DIR}/create-labguide-server.sh" && \
chmod +x "${INSTALL_DIR}"/*.sh && \
sudo "${INSTALL_DIR}/register-labguide-server.sh"; \
rm -rf "${INSTALL_DIR}"
```
