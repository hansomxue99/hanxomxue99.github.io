+++
title = "作业提交统计工具"
date = 2022-04-05T12:21:48+08:00
author = "hansomxue99"
keywords = ["python", "工具"]
cover = ""
summary = "基于python设计了一个简单的作业提交统计工具"

+++

## 做学委收一次作业真的不容易，为了提高效率，设计了一个简易的作业提交的统计工具。
功能描述：学委收到同学的作业（文件名要求：学号+姓名.pdf/doc/...），将这些文件统一放到一个directory文件夹下。同时需要有一份学生名单，包括（序号+姓名+学号），放在和directory同目录下，最后在这个目录下运行代码即可生成未提交作业的学生名单。
> 内容比较简单，开源共享！
```python
import xlrd
import xlsxwriter as xlwt
import os

_filename = 'excel.xlsx'
_dirname = 'directory'


def excel_list(filename):
    tabel = xlrd.open_workbook(filename)  # 得到excel表格
    sheet = tabel.sheet_by_index(4)  # 得到excel第一张sheet数据
    nrows = sheet.nrows  # 获得行数

    # 将excel中的数据生成列表
    list = []
    for i in range(2, nrows):
        list.append(sheet.row_values(i))
    return list, nrows - 2


def dir_list(dirname):
    dirs = os.listdir(dirname)
    list = []
    for file in dirs:
        if file.startswith('1'):
            index_start = file.find('1')
            index_stop = file.find('.')
            ID = file[index_start:index_start + 10]
            Name = file[index_start + 10:index_stop]
            list.append([Name, float(ID)])
    return list


if __name__ == '__main__':
    submitted = 0
    unsubmitted = 0
    listexcel, nrows = excel_list(_filename)
    listdir = dir_list(_dirname)
    for stnum in listdir:
        for i in listexcel:
            if stnum[0] == i[1]:
                submitted = submitted + 1
                listexcel.remove(i)
                continue
    unsubmitted = nrows - submitted
    print(submitted, unsubmitted)
    # print(listexcel)
    # 写入excel
    workbook = xlwt.Workbook(".\未提交作业名单.xlsx")  # 创建工作簿
    sheet1 = workbook.add_worksheet("未提交作业名单")  # 创建子表
    title = ['序号', '姓名', '学号']  # 设置表头
    sheet1.set_column('A:A', 5)  # 设置A列宽度30
    sheet1.set_column('B:B', 10)  # 设置B列宽度20
    sheet1.set_column('C:C', 20)  # 设置B列宽度20
    for i in range(len(title)):
        sheet1.write(0, i, title[i])
    for i in range(0, len(listexcel)):
        sheet1.write(i + 1, 0, i + 1)
        sheet1.write(i + 1, 1, listexcel[i][1])
        sheet1.write(i + 1, 2, str(int(listexcel[i][2])))
    workbook.close()

```
这次就分享这么多，感谢支持！