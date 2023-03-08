#Defensive #Windows #Tools 

# Script 1
- Password changes
- Add users that are in the csv file but not on the system (queries user first tho)
- Remove users that are on the system but not on the csv (queries user first tho)
- Update csv to include previously unlisted users

#### Usage 

In order to run the program you will use the following:
- `.\PasswordChanger.ps1 <csv file>`


###### CSV format
- "N/A" tells script that no password is supplied and should be marked as empty 
- first column stores the usernames 
- second column stores the passwords


```
user,password
bob,password123
alice,password123
Administrator,N/A
DefaultAccount,N/A
dunla,N/A
etand,N/A
Guest,N/A
WDAGUtilityAccount,N/A
```

###### Version 2
```powershell
param($file, $ShowError)



# Import csv file
$csvfile = Import-Csv -Path $file
$UsersToMake = @()
    
write-host "`n"

# Number of successfull password changes
$successes = $csvfile.Count


# determine if uer wants errors to be shown or not
if(!($ShowError -eq "--showerrors")){
    $ErrorActionPreference = 'silentlycontinue'
}

#import csv file with only names
$UsersOnly = Import-csv -Path $file | Select-Object -ExpandProperty user

#declare array to hold mismatched users
$mismatch = @()

#get the current list of users
$CurrentUsers = Get-LocalUser | Select-Object -ExpandProperty name



# for each row in csvfile
ForEach($i in $csvfile){
    
    #if user currently exists
    if(($CurrentUsers -contains $i.user) -and ($i.password -ne "N/A")){
        # perform password change
        Net User $i.user $i.password | Out-Null
        Add-LocalGroupMember -Group "Users" -Member $i.user | Out-Null

        write-host "Successfully" -foreground green -nonewline 
        write-host " updated the password for" $i.user
    }

    #if user has no password provided
    elseif($i.password -eq "N/A"){
    
        write-host "ERROR:" -foreground red -nonewline
        write-host " The user" $i.user "has no password supplied"
    }
    # else command failed or no user exists
    else{
        # print error message and decrement successes
        write-host "ERROR:" -foreground red -nonewline
        write-host " The user" $i.user "may not exist"
        $successes--
        $UsersToMake += $i
    }
    
}
    
# print out total number of successes 
write-host "`nSUCCESSFULLY" -foreground green -nonewline 
write-host " processed" $successes "out of" $csvfile.Count "users."


#if there exists users on the csv but not on the system
if($successes -ne $csvfile.Count){
    
    #for each non-existent user
    foreach($i in $UsersToMake){

        if(!($CurrentUsers -contains $i.user)){
            # query user choice
            $decision = read-host "`nWould you like to add the non-existent user" $i.user "to the system: y/n" 

            # if "y" or "Y"
            if(($decision -eq "y") -or ($decision -eq "Y")){
        
                        $secure = ConvertTo-SecureString -String $i.password -AsPlainText -Force
                        New-LocalUser $i.user -Password $secure | Out-Null
                        Add-LocalGroupMember -Group "Users" -Member $i.user | Out-Null
                }
              
            else{
                continue
            }
        }
    }
        

        write-host "`nSUCCESSFULLY" -foreground green -nonewline 
        write-host " processed the selected users."
}

    


#for every current user
foreach($element in $CurrentUsers){
    
    #check if they exist in the csv file and add to mismatched accordingly
    if(!($UsersOnly -contains $element)){
       $mismatch += $element
    }
    
}

#if there exists users on the system that are not documented on the csv
if($mismatch.Count -gt 0){
    
    write-host "`nSome users were found on the system that weren't specified in the csv file`n" -foreground yellow

    #for every mismatched user
    foreach($element in $mismatch){
    

        #query if you want to keep them
        $decision = read-host "Would you like to keep" $element "as a user? y/n"


        #if yes then add name to csv file
        if(($decision -eq "y") -or ($decision -eq "Y")){
    
            "`n{0},{1}" -f $element,"N/A" | add-content -path $file
        }

        #if no then remove them from the system
        else{
            Remove-LocalUser $element
            write-host "SUCCESSFULLY:" -foreground green -NoNewline
            write-host " removed"$element
        }
    }
}


write-host "`n`nALL OPERATIONS COMPLETE" -foreground green
```


###### Version 1


```powershell
param($verbose)

# Import csv file
$csvfile = Import-Csv -Path "C:\users\user\Documents\test.csv"
$UsersToMake = @()
    
    write-host "`n"

    # Number of successfull password changes
    $successes = $csvfile.Count

    # for each row in csvfile
    ForEach($i in $csvfile){

        # if "--verbose" was supplied as a command line argument
        if($verbose -eq "--verbose"){
            
            # perform password change
            Net User $i.user $i.password 2>$null | Out-Null

            # if last command was successfull
            if($?){
            write-host "Successfully updated the password for" $i.user
            }

            # else command failed
            else{
            # print error message and decrement successes
            write-host "ERROR:" -foreground red -nonewline
            write-host " The user" $i.user "may not exist"
            $successes--
            $UsersToMake += $i
            }
        }
        
        # if "--verbose" was not supplied as a command line argument
        else{
            
            # perform password change
            Net User $i.user $i.password 2>$null | Out-Null
            
            # if previous command failed
            if(!$?){
            # print error message and decrement successes
            write-host "ERROR:" -foreground red -nonewline 
            write-host " The user"$i.user"may not exist"
            $successes--
            $usersToMake += $i
            }
        }
    }
    
    # print out total number of successes 
    write-host "`nSUCCESSFULLY" -foreground green -nonewline 
    write-host " processed" $successes "out of" $csvfile.Count "users."

    if($successes -ne $csvfile.Count){
        # query user choice
        $decision = read-host "`nWould you like to add the non-existent users? y/n" 

        # if "y" or "Y"
        if(($decision -eq "y") -or ($decision -eq "Y")){
        
            # add each non-existent user
            ForEach($i in $UsersToMake){
            
                $secure = ConvertTo-SecureString -String $i.password -AsPlainText -Force
                New-LocalUser $i.user -Password $secure | Out-Null


            }

            write-host "`nSUCCESSFULLY" -foreground green -nonewline 
            write-host " processed the remaining" $UsersToMake.Count "users."
        }
    }

```
