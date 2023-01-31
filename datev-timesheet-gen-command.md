dayplan timesheet -f 2023-01-01 -t 2023-01-31 -i 'work' -e notrack --include-empty --duration-format "colon-delimited" \
  | sed 's/://g' \
  | ssconvert /dev/stdin ~/vm-shared/data.xls
