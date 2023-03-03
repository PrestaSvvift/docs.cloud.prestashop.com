---
title: Integrating a Module with PrestaShop CloudSync
---

# :arrows_counterclockwise: Integrating a Module with PrestaShop CloudSync

## Prerequisites

To complete this procedure, you need to have already integrated your module with [PrestaShop Account](../3-prestashop-account/README.md).

:::warning Contact Us
Our team is here to help you get started with the implementation of PrestaShop CloudSync. Please [click here](https://meetings.hubspot.com/esteban-martin3/prestashop-new-framework-integration-meeting) to set up a meeting with us before proceeding with the integration of your module.
:::

## Edit the <module_name>.php File

### Install PrestaShop EventBus

There is no dependency to add in the composer of your module to support PrestaShop EventBus and CloudSync features.

To download the PrestaShop EventBus module dependency, add the following highlighted contents to the `<module_name>.php` file:
    
```php{4,23,24,25,26,27,28,29,30,31,32,33}
<?php
// ...

use PrestaShop\PrestaShop\Core\Addon\Module\ModuleManagerBuilder;

if (!defined('_PS_VERSION_')) {
    exit;
}

class <module_name> extends Module

    //...

    public function install()
    {
        Configuration::updateValue('ITNEVERWORKS_LIVE_MODE', false);

        return parent::install() &&
            $this->registerHook('header') &&
            $this->registerHook('displayBackOfficeHeader') &&
            $this->getService('ps_accounts.installer')->install();

        /* CloudSync */
        $moduleManager = ModuleManagerBuilder::getInstance()->build();

        if (!$moduleManager->isInstalled("ps_eventbus")) {
            $moduleManager->install("ps_eventbus");
        } else if (!$moduleManager->isEnabled("ps_eventbus")) {
            $moduleManager->enable("ps_eventbus");
            $moduleManager->upgrade('ps_eventbus');
        } else {
            $moduleManager->upgrade('ps_eventbus');
        }

    }
```

### Add context for the CDC

To allow the merchant to share their data with your services, you have to pair your module with a Cross Domain Component (CDC). To do so, you need to expose some context to your configuration page using the `PresenterService` of PrestaShop EventBus.

1. Add the following highlighted contents to the `<module_name>.php` file:

  ```php{11,39,40,41,42,43,44,45,46,47,48,49,50,51}
  public function getContent()
    {
        /**
         * If values have been submitted in the form, process them.
         */
        if (((bool)Tools::isSubmit('submit<module_name>Module')) == true) {
            $this->postProcess();
        }

        $this->context->smarty->assign('module_dir', $this->_path);
        $moduleManager = ModuleManagerBuilder::getInstance()->build();

        $accountsService = null;

        try {
            $accountsFacade = $this->getService('ps_accounts.facade');
            $accountsService = $accountsFacade->getPsAccountsService();
        } catch (\PrestaShop\PsAccountsInstaller\Installer\Exception\InstallerException $e) {
            $accountsInstaller = $this->getService('ps_accounts.installer');
            $accountsInstaller->install();
            $accountsFacade = $this->getService('ps_accounts.facade');
            $accountsService = $accountsFacade->getPsAccountsService();
        }

        try {
            Media::addJsDef([
                'contextPsAccounts' => $accountsFacade->getPsAccountsPresenter()
                    ->present($this->name),
            ]);

            // Retrieve Account CDN
            $this->context->smarty->assign('urlAccountsCdn', $accountsService->getAccountsCdn());

        } catch (Exception $e) {
            $this->context->controller->errors[] = $e->getMessage();
            return '';
        }

        if ($moduleManager->isInstalled("ps_eventbus")) {
            $eventbusModule =  \Module::getInstanceByName("ps_eventbus");
            if (version_compare($eventbusModule->version, '1.9.0', '>=')) {

                $eventbusPresenterService = $eventbusModule->getService('PrestaShop\Module\PsEventbus\Service\PresenterService');

                $this->context->smarty->assign('urlCloudsync', "https://integration-assets.prestashop3.com/ext/cloudsync-merchant-sync-consent/latest/cloudsync-cdc.js");

                Media::addJsDef([
                    'contextPsEventbus' => $eventbusPresenterService->expose($this, ['info', 'modules', 'themes', 'orders'])
                ]);
            }
        }

        $output = $this->context->smarty->fetch($this->local_path.'views/templates/admin/configure.tpl');
        return $output;
    }
  ```

2. Edit the required consents according to your needs. You can use:

  - `info` (mandatory): Store technical data (PrestaShop version, PHP version, etc.)
  - `modules` (mandatory): List of modules installed on the store
  - `themes` (mandatory): List of themes installed on the store
  - `carts`: Shopping cart data of the store (current carts, abandoned carts, cart contents, product details)
  - `carriers`: Characteristics of the carriers offered by the merchant (shipping rates, shipping weight, countries, shipping zones)
  - `categories`: Product categories offered by the store
  - `orders`: Orders data (order ID, order content, order status and history)
  - `products`: Products offered by the store (products, variations, specific prices)
  - `taxonomies`: Enhanced categories specific to PrestaShop Facebook (advanced categories)
  - `currencies`: List of the store currencies and conversion rates
  - `customers`: Anonymized clients known by the store

3. Edit the CDC URL according to your needs. You can use:

  - **Integration**: https://integration-assets.prestashop3.com/ext/cloudsync-merchant-sync-consent/latest/cloudsync-cdc.js
  - **Preproduction**: https://preproduction-assets.prestashop3.com/ext/cloudsync-merchant-sync-consent/latest/cloudsync-cdc.js
  - **Production**: https://assets.prestashop3.com/ext/cloudsync-merchant-sync-consent/latest/cloudsync-cdc.js

## Edit the Template File

1. Access the template file allowing you to render the configuration page for your module in the back office (located by default at `views/templates/admin/configure.tpl`).

2. In the template file, add the following tag at the beginning:

  ```javascript
  <div id="prestashop-cloudsync"></div>
  ```

3. In the template file, add the following script lines at the end:

  ```javascript
  <script src="{$urlCloudsync|escape:'htmlall':'UTF-8'}"></script>

  <script>
      window?.psaccountsVue?.init();
		  // CloudSync
		  const cdc = window.cloudSyncSharingConsent;

	    cdc.init('#prestashop-cloudsync');
	    cdc.on('OnboardingCompleted', (isCompleted) => {
	      console.log('OnboardingCompleted', isCompleted);
	    });
	    cdc.isOnboardingCompleted((isCompleted) => {
	      console.log('Onboarding is already Completed', isCompleted);
	    });
  </script>
  ```

:::tip Note
If you prefer to set the rendering into another element, you can pass the `querySelector` to the init method as follows: `cdc.init("#consents-box")`
:::

:::tip Note
A callback function is available: it is called when the user gives their consent.
:::

## Test Your Module

To test if PrestaShop CloudSync is loading successfully into your module:

1. Zip your module folder.

2. In the back office of your PrestaShop store, go to **Modules** > **Module Catalog**.

3. Click the **Upload a module** button and select your archive.

4. Click **Configure** in the pop-up window that displays.
    
    :arrow_right: Your module configuration page should contain the PrestaShop CloudSync pane under the PrestaShop Account one:

    ![PrestaShop CloudSync pane](/assets/images/cloudsync/cloudsync-share-my-data.png)