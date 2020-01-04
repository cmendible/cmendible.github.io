---
author: Carlos Mendible
categories:
- Azure
- dotNet
date: "2010-10-18T10:21:03Z"
description: Prepare your App for Windows Azure! Create a custom Configuration Manager!
# tags: ["Configuration", "ServiceDefinition", "Windows"]
title: Prepare your App for Windows Azure! Create a custom Configuration Manager!
---
When you deploy an application to Windowes Azure, your application configuration cannot be changed cause it becomes part of the distributed package.

So what can you do if you want to set some dynamic settings (i.e Connection String settings)?

First of all you must use the **ServiceDefinition.csdef** file and add the dynamic settings there.

Next tell your app to read the connection string from that file instead of the config file. To accomplish that task create the following classes:

``` csharp 
namespace Azure.Configuration
{
    public class ConfigurationManager
    {
        private static Settings _settings = new Settings();
        public static Settings Settings
        {
            get
            {
                return _settings;
            }
        }

        public static ConnectionStrings ConnectionStrings
        {
            get
            {
                return new ConnectionStrings();
            }
        }
    }

    public class Settings
    {
        public string this[string key]
        {
            get
            {
                if (RoleEnvironment.IsAvailable)
                {
                    return RoleEnvironment.GetConfigurationSettingValue(key);
                }
                else
                {
                    if (cfg.ConfigurationManager.AppSettings.AllKeys.Contains(key))
                        return cfg.ConfigurationManager.AppSettings[key];
                }
                return string.Empty;
            }
        }
    }

    public class ConnectionStrings
    {
        public string this[string key]
        {
            get
            {
                if (RoleEnvironment.IsAvailable)
                {
                    return RoleEnvironment.GetConfigurationSettingValue(key);
                }
                else
                {
                    if (cfg.ConfigurationManager.ConnectionStrings[key] != null)
                        return cfg.ConfigurationManager.ConnectionStrings[key].ConnectionString;
                }
                return string.Empty;
            }
        }
    }
}
``` 

Then you can just call **ConfigurationManager.Settings[_{key}_]** to get the vaue for a given key or **ConfigurationManager.ConnectionStrings[_{connection string name}_]** to get a connection string.

You'll need to add a reference to **Microsoft.WindowsAzure.ServiceRuntime** which is included in the [Windows Azure Tools for Microsoft Visual Studio](http://www.microsoft.com/downloads/en/details.aspx?FamilyID=2274a0a8-5d37-4eac-b50a-e197dc340f6f&displaylang=en)

Hope it helps!