# PowerShell：遍历文件夹大小

```powershell
function filesize ([string]$filepath)
{
  if($filepath -eq $null){
    throw "null path"
  }
  dir -Path $filepath | ForEach-Object -Process{
    if($_.psiscontainer -eq $true){
      $length=0
      dir -Path $_.fullname -Recurse | ForEach-Object{
        $length+=$_.length
      }
      $l=$length/1GB
      $_.name+" size is : {0:n3} GB" -f $l
    }
  }
}
filesize -filepath "C:\"
```

