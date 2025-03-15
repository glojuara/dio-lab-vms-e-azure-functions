# Azure Deployment Guide

Este guia fornece um passo a passo completo para criar e gerenciar recursos no Azure usando arquivos ARM (Azure Resource Manager). Ele cobre a criação de:

- **Conjunto de Dimensionamento de Máquinas Virtuais (VMSS)** com Spot Instances
- **Aplicativo de Função (Function App)**
- **Clean up** para remover os recursos após o uso

> ✅ **Importante:** Este tutorial foi desenvolvido para ser executado diretamente no **Azure Cloud Shell**.

---

## 🚀 **1. Criação de Resource Group**

Antes de criar qualquer recurso, você deve criar um **Resource Group** para organizar os recursos associados.

```bash
az group create \
  --name "<RESOURCE_GROUP>" \
  --location "<REGION>"
```

| Parâmetro | Descrição |
|-----------|-----------|
| `<RESOURCE_GROUP>` | Nome do grupo de recursos |
| `<REGION>` | Região do Azure onde o grupo será criado (exemplo: `eastus`) |

---

## 🖥️ **2. Criação de Conjunto de Dimensionamento de Máquinas Virtuais (VMSS)**

### **2.1. Criação do Arquivo JSON para VMSS**
Crie o arquivo `vmss-template.json` usando o comando `echo`:

```bash
echo '{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.Compute/virtualMachineScaleSets",
      "apiVersion": "2023-01-01",
      "name": "[parameters('vmssName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "[parameters('vmSize')]",
        "tier": "Standard",
        "capacity": "[parameters('instanceCount')]"
      },
      "properties": {
        "upgradePolicy": {
          "mode": "Automatic"
        },
        "virtualMachineProfile": {
          "storageProfile": {
            "imageReference": {
              "publisher": "Canonical",
              "offer": "UbuntuServer",
              "sku": "18.04-LTS",
              "version": "latest"
            },
            "osDisk": {
              "createOption": "FromImage",
              "deleteOption": "Delete",
              "managedDisk": {
                "storageAccountType": "Standard_LRS"
              }
            }
          },
          "osProfile": {
            "computerNamePrefix": "[parameters('vmssName')]",
            "adminUsername": "[parameters('adminUsername')]",
            "adminPassword": "[parameters('adminPassword')]"
          },
          "networkProfile": {
            "networkInterfaceConfigurations": [
              {
                "name": "[concat(parameters('vmssName'), '-nic')]",
                "properties": {
                  "primary": true,
                  "ipConfigurations": [
                    {
                      "name": "[concat(parameters('vmssName'), '-ipconfig')]",
                      "properties": {
                        "subnet": {
                          "id": "[parameters('subnetId')]"
                        }
                      }
                    }
                  ]
                }
              }
            ]
          },
          "priority": "Spot",
          "evictionPolicy": "Delete"
        }
      }
    }
  ],
  "parameters": {
    "vmssName": {
      "type": "string"
    },
    "location": {
      "type": "string"
    },
    "vmSize": {
      "type": "string"
    },
    "instanceCount": {
      "type": "int"
    },
    "adminUsername": {
      "type": "string"
    },
    "adminPassword": {
      "type": "securestring"
    },
    "subnetId": {
      "type": "string"
    }
  }
}' > vmss-template.json
```

### **2.2. Deploy da VMSS**

```bash
az deployment group create \
  --name "DeployVMSS" \
  --resource-group "<RESOURCE_GROUP>" \
  --template-file "vmss-template.json" \
  --parameters vmssName="<VMSS_NAME>" location="<REGION>" vmSize="Standard_D2s_v3" instanceCount=2 adminUsername="<ADMIN_USER>" adminPassword="<ADMIN_PASSWORD>" subnetId="/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>/providers/Microsoft.Network/virtualNetworks/<VNET>/subnets/<SUBNET>"
```

---

## 🌐 **3. Criação de Function App**

### **3.1. Criação do Arquivo JSON para Function App**

```bash
echo '{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.Web/serverfarms",
      "apiVersion": "2023-01-01",
      "name": "[parameters('servicePlanName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Y1",
        "tier": "Dynamic"
      }
    },
    {
      "type": "Microsoft.Web/sites",
      "apiVersion": "2023-01-01",
      "name": "[parameters('functionAppName')]",
      "location": "[parameters('location')]",
      "kind": "functionapp",
      "properties": {
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', parameters('servicePlanName'))]"
      }
    }
  ]
}' > function-app-template.json
```

### **3.2. Deploy do Function App**

```bash
az deployment group create \
  --name "DeployFunctionApp" \
  --resource-group "<RESOURCE_GROUP>" \
  --template-file "function-app-template.json" \
  --parameters servicePlanName="<SERVICE_PLAN_NAME>" functionAppName="<FUNCTION_APP_NAME>" location="<REGION>"
```

---

## 🧹 **4. Cleanup - Removendo os Recursos Criados**

```bash
az group delete --name "<RESOURCE_GROUP>" --yes --no-wait
```

---

## ✅ **Resumo das Configurações**
✔️ Criação de Resource Group
✔️ Deploy de VMSS com Spot Instances
✔️ Deploy de Function App
✔️ Cleanup eficiente
