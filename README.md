## Creating a development environment from scratch on Ubuntu

1. Install curl and JRE

```shell
$ sudo apt install curl
$ sudo apt install openjdk-11-jre-headless
```

2. Install latest Kibana and ElasticSearch via apt

```shell
$ wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
$ echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic.list
$ sudo apt update
$ sudo apt install elasticsearch
$ sudo apt install kibana
```

For testing purpose, it may be required to install a specific (not latest) version of Kibana or ElasticSearch.

```shell
$ sudo apt install kibana    [for latest version]
$ sudo apt install kibana=7.8.0    [OR for a specific version]
```

3. Adjust listening IP address of Kibana if network access is required

```shell
$ sudo vi /etc/kibana/kibana.yml
  server.host: "192.168.1.77"    [update with correct IP value]
```

4. Start ElasticSearch, possibly upload the accounts test data

```shell
$ sudo systemctl start elasticsearch
$ curl -X GET "localhost:9200"
$ curl -H 'Content-Type: application/x-ndjson' -XPOST 'localhost:9200/bank/account/_bulk?pretty' --data-binary @accounts.json    [optional]
```

5. Now to create a Kibana development environment, download nvm, git client and yarn

```shell
$ curl https://raw.githubusercontent.com/creationix/nvm/v0.25.0/install.sh | bash    [then open a new terminal]
$ nvm install 10.21.0
$ sudo apt install git
$ curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
$ echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
$ sudo apt update
$ sudo apt install yarn
```

6. Download Kibana source code. After download, `kibana` is the top directory. Then select the target version by selecting a tag or a branch (v7.6.2, v7.8.0, v7.8, etc.)

```shell
$ git clone https://github.com/elastic/kibana.git
$ cd kibana
$ git checkout v7.8.0
```

7. Copy the source code of this plugin with modified name inside the `kibana/plugins` directory

8. Start Kibana in development mode, ensuring only OSS (Open Source Software) features are used. This step may take few minutes for the first compilation

```shell
$ cd kibana
$ nvm use 10.21.0    [or expected version. nvm install n.n.n may be required if version is missing]
$ yarn kbn bootstrap
$ yarn start --oss
```

9. Kernel values adjustment for large number of file monitoring may be required

```shell
$ echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf
$ sudo sysctl -p
```

10. If you have a problem to start a higher version of Kibana than ElasticSearch, with the error message during development`[error][savedobjects-service] This version of Kibana (v8.0.0) is incompatible with the following Elasticsearch nodes in your cluster: v7.8.0 @ 127.0.0.1:9200 (127.0.0.1)`, add this line in `config/kibana.yml` config file. As a side note when upgrading from v7.6.2 to v7.8 branch I had to delete all indexes `curl -XDELETE localhost:9200/*`

```
elasticsearch.ignoreVersionMismatch: true
```

11. When it is time to upgrade the Kibana development environment, start for a clean environment, get the latest changes from github, and switch to the new tag or release. The `kibana/plugins` directory remains untouched accross these steps. You may delete the `target` folder in each plugin folder.

```shell
$ cd kibana
$ git reset --hard
$ git fetch
$ git checkout v7.8.1
```

## Creating the actual form (step 7) for your own usecase

The current plugin name is based on the accounts test data. Simply perform a search replace in filenames and in the source code, respecting letter capitalization.

The form itself is contained in the [controller file](https://github.com/guyplusplus/Kibana-Plugin-Custom-Form-Filter-Visualization/blob/master/vis_type_custom_form_filter_accounts/public/custom_form_filter_accounts_vis_controller.tsx) file. An [option tab](https://github.com/guyplusplus/Kibana-Plugin-Custom-Form-Filter-Visualization/blob/master/vis_type_custom_form_filter_accounts/public/custom_form_filter_accounts_options.tsx) is also possible, actually one or more.

The form code looks like this and is very simple to modify, based on EUI React components.
* [EUI Documentation](https://elastic.github.io/eui/#/)
* [EUI GitHub repository](https://github.com/elastic/eui)

```xml
<EuiForm>
  <EuiFormRow label="Age" helpText="Input customer age">
    <EuiFieldText name="age" onChange={e => this.onFormChange(e)} value={this.state.age} />
  </EuiFormRow>
  <EuiFormRow label="Minimum balance" helpText={minimumBalanceHelpText} >
    <EuiFieldText name="minimumBalance" onChange={e => this.onFormChange(e)} value={this.state.minimumBalance} />
  </EuiFormRow>
  <EuiSpacer />
  <EuiText size="xs"><h4>State</h4></EuiText>
  <EuiComboBox
    placeholder="Select a state"
    isLoading={this.isLoading}
    singleSelection={{ asPlainText: true }}
    options={this.state.countryStates}
    selectedOptions={this.state.countryStateSelected}
    onChange={this.onCountryStateChange}
    isClearable={false}
  />
  <EuiSpacer />
  <EuiButton onClick={this.onClickButtonApplyFilter} fill>Apply filter</EuiButton>&nbsp;
  <EuiButton onClick={this.onClickButtonDeleteFilter} >Delete filter</EuiButton>&nbsp;
  <EuiButton onClick={this.onClickButtonClearForm} >Clear form</EuiButton>&nbsp;
  <EuiButton onClick={this.onClickButtonToday} color="secondary">Time: today</EuiButton>
</EuiForm>
```

I use [Microsoft Code](https://code.visualstudio.com/) to edit code and [Google Chrome](https://www.google.com/chrome/) to debug.

## Packaging the plugin as a zip file

Build the zip file with the `plugin_helpers.js` script

```
$ cd kibana/plugins/vis_type_custom_form_filter_accounts
$ node ../../scripts/plugin_helpers.js build
? What version of Kibana are you building for? 7.10.0
 info deleting the build and target directories
 info running @kbn/optimizer
 │ info initialized, 0 bundles cached
 │ info starting worker [1 bundle]
 │ succ 1 bundles compiled successfully after 45 sec
 info copying source into the build and converting with babel
 info compressing plugin into [visTypeCustomFormFilterAccounts-7.10.0.zip]
$
```

## Installing the plugin

The plugin can then be installed like this for an apt installed Kibana.

```shell
$ cd /usr/share/kibana
$ sudo -u kibana ./bin/kibana-plugin install file:///home/john/downloads/vis_type_custom_form_filter_accounts_7.9.0_1.0.2.zip
$ sudo -u kibana ./bin/kibana-plugin install https://github.com/guyplusplus/Kibana-Plugin-Custom-Form-Filter-Visualization/releases/download/7.9.0-1.0.2/vis_type_custom_form_filter_accounts_7.9.0_1.0.2.zip
```

Deleting then installing the plugin often fails for me. I fix it by running this command.

```shell
$ cd /usr/share/kibana
$ sudo -u kibana ./bin/kibana-plugin remove visTypeCustomFormFilterAccounts
Removing visTypeCustomFormFilterAccounts...
Plugin removal complete
$ sudo rm -rf /usr/share/kibana/optimize/bundles
$
```

## Project TODO List

- [X] Create form content (i.e. dropdown, slider) with actual data
- [X] Sample code to modify time filter
- [ ] Create a script to replace 'accounts' in filenames and file content
- [ ] Add internationalization example
- [ ] Create test script
- [X] Create own plugin icon
- [ ] Improve fetchData (actually Array) with a more reusable and paramerizable API


