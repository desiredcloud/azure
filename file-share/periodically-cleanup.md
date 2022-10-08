
##### Delete files from Azure File Share periodically after $N of Days


```yaml
# Python package
# Create and test a Python package on multiple Python versions.
# Add steps that analyze code, save the dist with the build record, publish to a PyPI-compatible index, and more:
# https://docs.microsoft.com/azure/devops/pipelines/languages/python

trigger:
- none

variables:
  account_name: '"account-1"'
  account_key: '"abcef..............."'
  share_name: '"share-1"'
  directory_name: '""'
  if_older_than: 7

pool:
  vmImage: 'ubuntu-20.04'
strategy:
    matrix:
      Python38:
        PYTHON_VERSION: '3.8'
    maxParallel: 3

steps:
- task: UsePythonVersion@0
  inputs:
    versionSpec: '$(PYTHON_VERSION)'
    architecture: 'x64'
  displayName: 'Use Python $(python.version)'

- script: |
    python -m pip install --upgrade pip setuptools wheel
    pip install azure-common
    pip install azure-storage-common
    pip install azure-storage-file
    pip install azure-storage-file-share
    pip freeze
    python -c "from azure.storage.file import FileService; print(FileService)"
  displayName: 'Install dependencies'


- task: PythonScript@0
  inputs:
    scriptSource: 'inline'
    script: |
      from _datetime import datetime
      from azure.storage.file import FileService

      """
      This script is to periodically cleanup testautomation's result in ihcpdev's fileshare
      """


      def test_result_periodic_cleanup(account_name, account_key, share_name, directory_name, if_older_than):
          '''
          :param str account_name: The storage account name.
          :param str account_key: The storage account key. This is used for shared key authentication.
          :param str share_name: Name of existing share.
          :param str directory_name: The path to the directory.
          :param int if_older_than: will delete file older than provided number of days.
          '''

          try:
              file_service = FileService(account_name=account_name, account_key=account_key)
              files = file_service.list_directories_and_files(share_name)
          except Exception as e:
              print("Failed to list files from file share: {share_name} of storage account: {account_name}".format(
                  share_name=share_name, account_name=account_name), e)
              return

          if files:
              for file in files:
                  try:
                      f_properties = file_service.get_file_properties("ihcpdev-test-results", "", file.name)
                      datetime_now = datetime.now()
                      is_older_than = datetime_now - f_properties.properties.last_modified.replace(tzinfo=None)
                      if is_older_than.days > if_older_than:
                          print("Deleting file name: {name}, lastModified: {last_modified}, created: {is_older_than} days ago".format(name=f_properties.name,
                                                                        last_modified=f_properties.properties.last_modified, is_older_than=is_older_than.days))
                          resp = file_service.delete_file(share_name, directory_name, f_properties.name)
                          print("delete response: ", resp)
                  except Exception as e:
                      print("Failed to list files from file share: {share_name} of storage account: {account_name}".format(
                          share_name=share_name, account_name=account_name), e)
          else:
              print("No files found in file share: {share_name} of storage account: {account_name}".format(
                  share_name=share_name, account_name=account_name))


      if __name__ == '__main__':
          test_result_periodic_cleanup($(account_name), $(account_key), $(share_name), $(directory_name), $(if_older_than))

  displayName: 'Executing Script'
```