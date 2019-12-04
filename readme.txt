Function Install-X509CertificateToJavaTrustedCaKeystores {
  [CmdletBinding()]
  Param(
    [Parameter(Mandatory = $True, ValueFromPipelineByPropertyName=$True, Position = 0)]
    [ValidateNotNullOrEmpty()]
    [Alias("CertificateObject", "X509CertificateObject", "X509Certificate2")]
    [String] $X509Certificate,
    [Parameter(Mandatory = $False, ValueFromPipelineByPropertyName=$True, Position = 1)]
    [Alias("FriendlyName", "Name", "Alias", "CertificateAlias")]
    [String] $CertAlias,
    [Parameter(Mandatory = $True, Position = 3)]
    [ValidateNotNullOrEmpty()]
    [Alias("StorePassword", "StorePass", "Password", "Pass")]
    [String] $KeyStorePass = "changeit"
  )

  BEGIN {

    # Gather all java keystores - this is only required once when running multiple times
    $JavaKeystores = Get-JavaCertStores("c:\Program Files\Android\Android Studio1")
  }

  PROCESS {

    

    Try {
      # Import the certificate to all given java keystores
      $JavaKeyStores | Import-TrustedCaCertificateToJavaKeystore -CertAlias $CertAlias -CertFile $X509Certificate -KeyStorePass $KeyStorePass
    }
    Finally {
      # Remove the exported certificate after importing to all stores
      Remove-Item $TempCertFile
    }

  }

}


Function Get-JavaCertStores($path){

If (!(Test-Path $path)) {
    return "Directory not found"
}
 


 $KeyTool = $(Join-Path -Path $path -ChildPath "jre/bin/keytool.exe")
    If (!(Test-Path -Path $KeyTool -PathType Leaf)) { $KeyTool = "keytool" }

    # find existing cacerts files at different locations within java home
    $KeyStores = @("jre/jre/lib/security/cacerts") |
      Foreach-Object { Join-Path -Path $path -ChildPath $_ } |
      Where-Object { Test-Path -Path $_ -PathType Leaf }

    # return PSCustomObject with keystore and keytool locations
   $KeyStores | ForEach-Object { [PSCustomObject] @{
      KeyStore = $_
      KeyTool = $KeyTool
    }}
  



}




Function Import-TrustedCaCertificateToJavaKeystore {
  [CmdletBinding()]
  Param(
    [Parameter(Mandatory = $True, Position = 0)]
    [ValidateNotNullOrEmpty()]
    [Alias("File", "Certificate", "CertificateFile")]
    [String] $CertFile,
    [Parameter(Mandatory = $True, Position = 1)]
    [ValidateNotNullOrEmpty()]
    [Alias("FriendlyName", "Name", "Alias", "CertificateAlias")]
    [String] $CertAlias,
    [Parameter(Mandatory = $True, ValueFromPipelineByPropertyName=$True, Position = 2)]
    [ValidateNotNullOrEmpty()]
    [Alias("Store", "CertificateStore", "cacerts")]
    [String] $KeyStore,
    [Parameter(Mandatory = $True, ValueFromPipelineByPropertyName=$True, Position = 3)]
    [ValidateNotNullOrEmpty()]
    [Alias("StorePassword", "StorePass", "Password", "Pass")]
    [String] $KeyStorePass = "changeit",
    [Parameter(Mandatory = $True, ValueFromPipelineByPropertyName=$True, Position = 4)]
    [ValidateNotNullOrEmpty()]
    [Alias("JavaKeyTool", "Command")]
    [String] $KeyTool = "keytool"
  )

  If (Test-CertificateExistsWithinJavaKeystore -CertAlias $CertAlias -KeyStore $KeyStore -KeyStorePass $KeyStorePass -KeyTool $KeyTool) {
    Write-Verbose ("Certificate with the alias {0} already exists in {1}." -f $CertAlias, $KeyStore)
    return
  }

  Add-TrustedCaCertificateToJavaKeystore -CertFile $CertFile -CertAlias $CertAlias -KeyStore $KeyStore -KeyStorePass $KeyStorePass -KeyTool $KeyTool

}


# Add certificate to java keystore
Function Add-TrustedCaCertificateToJavaKeystore {
  [CmdletBinding()]
  Param(
    [Parameter(Mandatory = $True, Position = 0)]
    [ValidateNotNullOrEmpty()]
    [Alias("File", "Certificate", "CertificateFile")]
    [String] $CertFile,
    [Parameter(Mandatory = $True, Position = 1)]
    [ValidateNotNullOrEmpty()]
    [Alias("FriendlyName", "Name", "Alias", "CertificateAlias")]
    [String] $CertAlias,
    [Parameter(Mandatory = $True, Position = 2)]
    [ValidateNotNullOrEmpty()]
    [Alias("Store", "CertificateStore", "cacerts")]
    [String] $KeyStore,
    [Parameter(Mandatory = $True, Position = 3)]
    [ValidateNotNullOrEmpty()]
    [Alias("StorePassword", "StorePass", "Password", "Pass")]
    [String] $KeyStorePass,
    [Parameter(Mandatory = $True, Position = 4)]
    [ValidateNotNullOrEmpty()]
    [Alias("JavaKeyTool", "Command")]
    [String] $KeyTool
  )

  $KeyToolCommand = Get-Command "$KeyTool" -ErrorAction Stop

  Try {
    $ExecutionResult = $(& $KeyToolCommand -import -noprompt -trustcacerts -keystore "$KeyStore" -alias "$CertAlias" -file "$CertFile" -storepass "$KeyStorePass")
  }
  Catch {
    Throw ("Encountered problems while trying to import trusted root certificate with alias: {0} to java certificate store {1} - Exception: {2}" -f $CertAlias, $KeyStore, $_.Exception.Message)
  }

  If ($LastExitCode -ne 0) {
    Write-Error ("Unable to import certificate with given alias: {0} to {1}. Keytool Output: `n{2}" -f $CertAlias, $KeyStore, $ExecutionResult)
  }

}


# Test if a certificate with the given alias exists in a java keystore
Function Test-CertificateExistsWithinJavaKeystore {
  [CmdletBinding()]
  Param(
    [Parameter(Mandatory = $True, Position = 0)]
    [ValidateNotNullOrEmpty()]
    [Alias("FriendlyName", "Name", "Alias", "CertificateAlias")]
    [String] $CertAlias,
    [Parameter(Mandatory = $True, Position = 1)]
    [ValidateNotNullOrEmpty()]
    [Alias("Store", "CertificateStore", "cacerts")]
    [String] $KeyStore,
    [Parameter(Mandatory = $True, Position = 2)]
    [ValidateNotNullOrEmpty()]
    [Alias("StorePassword", "StorePass", "Password", "Pass")]
    [String] $KeyStorePass,
    [Parameter(Mandatory = $True, Position = 3)]
    [ValidateNotNullOrEmpty()]
    [Alias("JavaKeyTool", "Command")]
    [String] $KeyTool
  )

  $KeyToolCommand = Get-Command "$KeyTool" -ErrorAction Stop

  Try {
    $ExecutionResult = $(& $KeyToolCommand -list -noprompt -keystore "$KeyStore" -alias "$CertAlias" -storepass "$KeyStorePass")
  }
  Catch {
    Throw ("Encountered problems while trying to check if given alias: {0} is already present in java certificate store {1} - Exception: {2}" -f $CertAlias, $KeyStore, $_.Exception.Message)
  }

  If ($LastExitCode -eq 0) { return $True }

  Write-Verbose ("Unable to find a certificate with given alias: {0} in {1}. Keytool Output: `n{2}" -f $CertAlias, $KeyStore, $ExecutionResult)
  return $False

}




<#check cert 
run it

Install-X509CertificateToJavaTrustedCaKeystores -X509Certificate "C:\Users\Roman\Downloads\CaCert.cer" -CertAlias "DigiCert Global Root CA"
#>
