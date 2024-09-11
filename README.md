
$originalString = ""
$regex = '<(\w+),\s*(\w+)\(([\d,]*)\)\s*,\s*>'
$replacementValues = @{}
$fieldsOrder = @()
function Generate-RandomString {
    param (
        [int]$length
    )
    $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'
    $random = New-Object System.Random
    -join ((1..$length) | ForEach-Object { $chars[$random.Next($chars.Length)] })
}
function Generate-SampleValue {
    param (
        [string]$dataType,
        [string]$lengthOrScale
    )
    if ($dataType -eq 'NCHAR' -or $dataType -eq 'VARCHAR') {
        $length = [int]$lengthOrScale
        $value = Generate-RandomString -length $length
        return "'$value'"  # Add single quotes for CHAR and VARCHAR types
    }
    elseif ($dataType -eq 'NUMERIC') {
        $parts = $lengthOrScale -split ','
        $precision = [int]$parts[0]
        $scale = [int]$parts[1]
        $maxValue = [math]::Pow(10, $precision - $scale) - 1
        return [math]::Round((Get-Random -Minimum 0 -Maximum $maxValue) + [math]::Pow(10, -$scale) * (Get-Random -Minimum 0 -Maximum 99), $scale)
    }
    elseif ($dataType -eq 'DATETIME2') {
        return (Get-Date).AddDays(-1 * (Get-Random -Minimum 1 -Maximum 365)).ToString('yyyy-MM-ddTHH:mm:ss')
    }
    return 'defaultValue'
}

$matches = [regex]::Matches($originalString, $regex)
$modifiedStrings = @()
$allReplacementValues = @()

$numberOfCopies = 3

foreach ($match in $matches) {
    $field = $match.Groups[1].Value
    $dataType = $match.Groups[2].Value
    $lengthOrScale = $match.Groups[3].Value
    $fieldsOrder += [PSCustomObject]@{
        Index = $fieldsOrder.Count
        Field = $field
        DataType = $dataType
        LengthOrScale = $lengthOrScale
    }
}

for ($i = 1; $i -le $numberOfCopies; $i++) {
    $modifiedString = $originalString
    $replacementValues = @{}
    foreach ($fieldInfo in $fieldsOrder) {
        $field = $fieldInfo.Field
        $dataType = $fieldInfo.DataType
        $lengthOrScale = $fieldInfo.LengthOrScale
        $sampleValue = Generate-SampleValue -dataType $dataType -lengthOrScale $lengthOrScale
        $replacementValues[$field] = $sampleValue
    }
    foreach ($fieldInfo in $fieldsOrder) {
        $field = $fieldInfo.Field
        $replacementValue = $replacementValues[$field]
        $pattern = [regex]::Escape("<$field, $($fieldInfo.DataType)($($fieldInfo.LengthOrScale)),>")
        $modifiedString = $modifiedString -replace $pattern, $replacementValue
    }
    $defaultValues = @{
        'NY_KS_KYAKUSU' = '1234567'
        'NY_KS_ZEIKIN' = "'987654321'"
    }
    foreach ($field in $defaultValues.Keys) {
        if ($replacementValues.ContainsKey($field)) {
            $replacementValues[$field] = $defaultValues[$field]
            $pattern = [regex]::Escape("<$field, $($fieldsOrder | Where-Object { $_.Field -eq $field }).DataType>($($fieldsOrder | Where-Object { $_.Field -eq $field }).LengthOrScale),>")
            $modifiedString = $modifiedString -replace $pattern, $defaultValues[$field]
        }
    }
    $modifiedStrings += $modifiedString
    $orderedValues = $fieldsOrder | Sort-Object Index | ForEach-Object { $replacementValues[$_.Field] }
    $allReplacementValues += ($orderedValues -join ',')
}
$outputFilePath = "D:\testToll\output.sql"
Set-Content -Path $outputFilePath -Value $null
for ($i = 0; $i -lt $numberOfCopies; $i++) {
    # Format the replacement values as a comma-separated list
    $replacementValuesContent = "Replacement Values Version $($i+1):`n($($allReplacementValues[$i]))"
    $replacementValuesContent | Out-File -FilePath $outputFilePath -Append
}

Write-Output "Data has been written to $outputFilePath"
