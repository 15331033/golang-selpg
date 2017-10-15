[TOC]
####golang开发Linux命令行实用程序selpg
#####1.设计说明
1.引入的包
```
import (
	"bufio"
	"fmt"
	"io"
	"os"
	"strconv"
)
```
1.结构体selpg_args
用来存储命令要求打印的开始页，结束页，输入文件名，页长，页类型和输出目的
如图所示
```
type selpg_args struct {
	start_page  int
	end_page    int
	in_filename string
	page_len    int
	page_type   int

	print_dest string
}
```
2.四个函数
分别是
(1)func usage()
输出正确的命令的各个参数
```
func usage() {
	fmt.Printf("\nUSAGE: %s -sstart_page -eend_page [ -f | -llines_per_page ]\n [ -ddest ] [ in_filename ]\n", progname)
}
```
(2)func process_args(ac int, av []string, psa *selpg_args)
对命令行参数的识别，并对该命令对应的结构体的参数进行赋值
```
func process_args(ac int, av []string, psa *selpg_args) {
	var s1 string
	var s2 string
	var argno int

	var i int
	//参数小于3则参数不够
	if ac < 3 {
		fmt.Printf("%s: not enough arguments\n", progname)
		usage()
		os.Exit(1)
	}
	//获取开始页
	s1 = av[1]
	if s1[0] != '-' || s1[1] != 's' {
		fmt.Printf("%s: 1st arg should be -sstart_page\n", progname)
		usage()
		os.Exit(2)
	}

	i, _ = strconv.Atoi(s1[2:])
	const maxInt = 1<<32 - 1
	if i < 1 || i > maxInt {
		fmt.Printf("%s: invalid start page %s\n", progname, s1[2:])
		usage()
		os.Exit(3)
	}
	psa.start_page = i
	//获取结束页
	s1 = av[2]
	if s1[0] != '-' || s1[1] != 'e' {
		fmt.Printf("%s: 2st arg should be -eend_page\n", progname)
		usage()
		os.Exit(4)
	}
	i, _ = strconv.Atoi(s1[2:])
	if i < 1 || i > maxInt || i < psa.start_page {
		fmt.Printf("%s: invalid end page %s\n", progname, s1[2:])
		usage()
		os.Exit(5)
	}
	psa.end_page = i
	//判断l参数，f参数，d参数
	argno = 3
	for argno <= ac-1 && av[argno][0] == '-' {
		s1 = av[argno]
		switch s1[1] {
		case 'l':
			s2 = s1[2:]
			i, _ = strconv.Atoi(s2)
			if i < 1 || i > maxInt {
				fmt.Printf("%s: invalid page length %s\n", progname, s2)
				usage()
				os.Exit(6)
			}
			//获得自定义的页长度
			psa.page_len = i
			argno += 1
		case 'f':
			if s1[0] != '-' || s1[1] != 'f' {
				fmt.Printf("%s: option should be \"-f\"\n", progname)
				usage()
				os.Exit(7)
			}
			//规定页类型，判断页结束符
			psa.page_type = 'f'
			argno += 1
		case 'd':
			s2 = s1[2:]
			if len(s2) < 1 {
				fmt.Printf("%s: -d option requires a printer destination\n", progname)
				usage()
				os.Exit(8)
			}
			psa.print_dest = s2
			argno += 1
		default:
			fmt.Printf("%s: unknown option %s\n", progname, s1)
			usage()
			os.Exit(9)

		}
	}
	//若还有参数，则是输入文件的文件名
	if argno <= ac-1 {
		psa.in_filename = av[argno]

		_, err := os.Stat(psa.in_filename)
		if os.IsNotExist(err) {
			fmt.Printf("%s: input file \"%s\" does not exist\n", progname, psa.in_filename)
			os.Exit(10)
		}
	}
	//判断各种异常条件
	if psa.start_page <= 0 || psa.end_page <= 0 || psa.end_page < psa.start_page || psa.page_len <= 1 || (psa.page_type != 'l' && psa.page_type != 'f') {
		os.Exit(11)
	}
}
```
(3)func process_input(sa selpg_args)
对输入输出根据结构体中的数据进行处理打印
```
func process_input(sa selpg_args) {
	var fin *os.File
	var fout *os.File
	var s1 string
	//var crc string
	//var c int
	//var line string
	var line_ctr int
	var page_ctr int
	//var inbuf string
	//判断是否有输入文件名，无就选择标准输入，否则打开文件
	if sa.in_filename[0] == '~' {
		fin = os.Stdin
	} else {
		fin, _ = os.Open(sa.in_filename)
		if fin == nil {
			fmt.Printf("%s: could not open input file \"%s\"\n", progname, sa.in_filename)
			os.Exit(12)
		}
	}
	//判断是否有输出文件名，无选择标准输出，否则打开文件
	if sa.print_dest[0] == '~' {
		fout = os.Stdout
	} else {

		s1 = sa.print_dest
		fout, _ = os.Open(s1)
		if fout == nil {
			fmt.Printf("%s: could not open pipe to \"%s\"\n", progname, s1)
			os.Exit(13)
		}
	}
	//若是l类型，则根据页长度打印相应的页数
	if sa.page_type == 'l' {
		line_ctr = 0
		page_ctr = 1
		rd := bufio.NewReader(fin)
		for {
			//读取一行
			line, ere := rd.ReadString('\n')
			if ere != nil {
				break
			}
			line_ctr += 1
			//若行数大于页长，页数加1，行数置1
			if line_ctr > sa.page_len {
				page_ctr += 1
				line_ctr = 1
			}
			//根据页数判断是否输出
			if page_ctr >= sa.start_page && page_ctr <= sa.end_page {
				_, _ = io.WriteString(fout, line)
			}

		}
	} else {
		//若是f类型
		page_ctr = 1
		rd := bufio.NewReader(fin)
		for {
			//读取每一行
			line, ere := rd.ReadString('\n')
			if ere != nil {
				break
			}
			//遍历每一行，若遇到'\f'，页数加1
			for _, v := range line {
				if v == '\f' {
					page_ctr += 1
				}
				//根据页数判断是否输出
				if page_ctr >= sa.start_page && page_ctr <= sa.end_page {
					_, _ = io.WriteString(fout, string(v))
				}
			}
		}
	}

	if page_ctr < sa.start_page {
		fmt.Printf("%s: start_page (%d) greater than total pages (%d), no output written\n", progname, sa.start_page, page_ctr)
	} else if page_ctr < sa.end_page {
		fmt.Printf("%s: end_page (%d) greater than total pages (%d), less output than expected\n", progname, sa.end_page, page_ctr)
	}
	fin.Close()
	fout.Close()
}
```
(4)func main()
初始化命令行输入的参数和命令要求打印的结构体sa
```
func main() {
	//获得参数
	ac := len(os.Args)
	av := os.Args

	var sa selpg_args

	progname = av[0]
	//初始化sa
	sa.start_page, sa.end_page = -1, -1
	sa.in_filename = "~"
	sa.page_len = 72
	sa.page_type = 'l'
	sa.print_dest = "~"
	//运行参数处理函数和输入处理函数
	process_args(ac, av, &sa)
	process_input(sa)
}
```




