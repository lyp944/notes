# 使用otool进行iOS应用瘦身
### 原理
使用Xcode内置的otool工具分析可执行文件Mach-o，在通过正则塞选过滤
### 使用
### 怎么查找的无用的类及方法
比如脚本保存为"objcthin.ruby"（脚本在最下边 主要来之于一个 gem 叫 objcthin，修改了一个正则的问题及一些补充），结果输入到“unused.text”

```shell
ruby objcthin.ruby "mach-o路径" "MG"(MG为类前缀可以不填) > unused.text
```
根据“unused.text”人工校验删除无用类及方法后，在用FengNiao（https://github.com/onevcat/FengNiao） 删除无用的图片
记得备份！记得备份！记得备份！
#### 找到Mach-o中的Objc segment段

```shell
otool -arch 'arm64' -V -o #{Mach-o Path}
```
输出中可以看到所有的objc类及结构，如：

```objc
/Users/mega/Desktop/mach-o test/MachOTestResult/MachOTest-debug:
//数据段 中的 __objc_classlist section
Contents of (__DATA,__objc_classlist) section 
//以下是类的基本信息
00000001000080f8 0x1000096f8 _OBJC_CLASS_$_ViewController   //类型
           isa 0x100009720 _OBJC_METACLASS_$_ViewController //isa指向
    superclass 0x0 _OBJC_CLASS_$_UIViewController           //super指向
         cache 0x0 __objc_empty_cache
        vtable 0x0
          data 0x1000081a8 (struct class_ro_t *)
                    flags 0x80
            instanceStart 8
             instanceSize 8
                 reserved 0x0
               ivarLayout 0x0
                     name 0x10000740c ViewController
              baseMethods 0x100008188 (struct method_list_t *)
		   entsize 24
		     count 1
		      name 0x1000065ac viewDidLoad
		     types 0x10000748b v16@0:8
		       imp 0x100005aac -[ViewController viewDidLoad]
            baseProtocols 0x0
                    ivars 0x0
           weakIvarLayout 0x0
           baseProperties 0x0
Meta Class
           isa 0x0 _OBJC_METACLASS_$_NSObject
    superclass 0x0 _OBJC_METACLASS_$_UIViewController
         cache 0x0 __objc_empty_cache
        vtable 0x0
          data 0x100008140 (struct class_ro_t *)
                    flags 0x81 RO_META
            instanceStart 40
             instanceSize 40
                 reserved 0x0
               ivarLayout 0x0
                     name 0x10000740c ViewController
              baseMethods 0x0 (struct method_list_t *)
            baseProtocols 0x0
                    ivars 0x0
           weakIvarLayout 0x0
           baseProperties 0x0

		......中间忽略了若干......

//数据段 中的 __objc_classrefs section 被引用的类
Contents of (__DATA,__objc_classrefs) section 
00000001000096b8 0x0 _OBJC_CLASS_$_UIColor
00000001000096c0 0x100009748 _OBJC_CLASS_$_UnusedClass0
00000001000096c8 0x0 _OBJC_CLASS_$_UISceneConfiguration
00000001000096d0 0x1000097c0 _OBJC_CLASS_$_AppDelegate
Contents of (__DATA,__objc_superrefs) section
00000001000096d8 0x1000096f8 _OBJC_CLASS_$_ViewController
00000001000096e0 0x100009748 _OBJC_CLASS_$_UnusedClass0
Contents of (__DATA,__objc_protolist) section
0000000100008118 0x100009840 __OBJC_PROTOCOL_$_NSObject
0000000100008120 0x1000098a0 __OBJC_PROTOCOL_$_UIApplicationDelegate
0000000100008128 0x100009900 __OBJC_PROTOCOL_$_UISceneDelegate
0000000100008130 0x100009960 __OBJC_PROTOCOL_$_UIWindowSceneDelegate
Contents of (__DATA,__objc_imageinfo) section
  version 0
    flags 0x40 OBJC_IMAGE_HAS_CATEGORY_CLASS_PROPERTIES
```

#### 通过正则找到所有累名及被引用的类名

```text
1. “Contents of (__DATA,__objc_classlist) section”  下
所有 “00000001000080f8 0x1000096f8 _OBJC_CLASS_$_ViewController” 行中的类名
2. 去掉 ”Contents of (__DATA,__objc_classrefs) section“ 下
所有 “00000001000096c0 0x100009748 _OBJC_CLASS_$_UnusedClass0” 行的类名就是没有被引用的类
```

#### 分析上边Objc segment段,同样的方式可以找到所有实现的方法(在类的结构里边)，但是仔细看上边并没有像`__DATA __objc_classrefs`类似的`__DATA __objc_selrefs` section 来找到引用的方法，看otool的所有命令中有一个"-l print the load commands"选项可以输出mach-o加载的所有命令

```shell
-f print the fat headers
  -a print the archive header
  -h print the mach header
  -l print the load commands
  -L print shared libraries used
  -D print shared library id name
  -t print the text section (disassemble with -v)
      ...
```

#### 发现‘-l’输出的load command 中有 “__DATA __objc_selrefs”的影子，之后在通过 -s 命令，再通过正则就可以找到引用的方法了

### 查找objc无用类的ruby脚本，脚本代码如下(代码来自于一个 gem 叫 objcthin)，代码时间比较早，导致查找无用方法的正则失效了，这个脚本主要是修复了那个正则问题

```ruby
require 'singleton'
require 'pathname'
require 'rainbow'

module Imp
  class UnusedSel

    include Singleton

    def find_unused_sel(path, prefix)
      check_file_type(path)
      all_sels = find_impl_methods(path)
      used_sel = reference_selectors(path)

      unused_sel = []

      all_sels.each do |sel,class_and_sels|
        unless used_sel.include?(sel)
          unused_sel += class_and_sels
        end
      end

      puts Rainbow('below selector is unused:').red
      if prefix
        unused_sel.select! do |classname_selector|
          current_prefix = classname_selector.byteslice(2, prefix.length)
          current_prefix == prefix
        end
      end

      puts unused_sel
    end

    def check_file_type(path)
      pathname = Pathname.new(path)
      # unless pathname.exist?
      #   raise "#{path} not exit!"
      # endc

      cmd = "/usr/bin/file -b #{path}"
      output = `#{cmd}`

      unless output.include?('Mach-O')
        raise 'input file not mach-o file type'
      end
      puts Rainbow('will begin process...').green
      pathname
    end

    def find_impl_methods(path)
      apple_protocols = [
          'tableView:canEditRowAtIndexPath:',
          'commitEditingStyle:forRowAtIndexPath:',
          'tableView:viewForHeaderInSection:',
          'tableView:cellForRowAtIndexPath:',
          'tableView:canPerformAction:forRowAtIndexPath:withSender:',
          'tableView:performAction:forRowAtIndexPath:withSender:',
          'tableView:accessoryButtonTappedForRowWithIndexPath:',
          'tableView:willDisplayCell:forRowAtIndexPath:',
          'tableView:commitEditingStyle:forRowAtIndexPath:',
          'tableView:didEndDisplayingCell:forRowAtIndexPath:',
          'tableView:didEndDisplayingHeaderView:forSection:',
          'tableView:heightForFooterInSection:',
          'tableView:shouldHighlightRowAtIndexPath:',
          'tableView:shouldShowMenuForRowAtIndexPath:',
          'tableView:viewForFooterInSection:',
          'tableView:willDisplayHeaderView:forSection:',
          'tableView:willSelectRowAtIndexPath:',
          'willMoveToSuperview:',
          'scrollViewDidEndScrollingAnimation:',
          'scrollViewDidZoom',
          'scrollViewWillEndDragging:withVelocity:targetContentOffset:',
          'searchBarTextDidEndEditing:',
          'searchBar:selectedScopeButtonIndexDidChange:',
          'shouldInvalidateLayoutForBoundsChange:',
          'textFieldShouldReturn:',
          'numberOfSectionsInTableView:',
          'actionSheet:willDismissWithButtonIndex:',
          'gestureRecognizer:shouldReceiveTouch:',
          'gestureRecognizer:shouldRecognizeSimultaneouslyWithGestureRecognizer:',
          'gestureRecognizer:shouldReceiveTouch:',
          'imagePickerController:didFinishPickingMediaWithInfo:',
          'imagePickerControllerDidCancel:',
          'animateTransition:',
          'animationControllerForDismissedController:',
          'animationControllerForPresentedController:presentingController:sourceController:',
          'navigationController:animationControllerForOperation:fromViewController:toViewController:',
          'navigationController:interactionControllerForAnimationController:',
          'alertView:didDismissWithButtonIndex:',
          'URLSession:didBecomeInvalidWithError:',
          'setDownloadTaskDidResumeBlock:',
          'tabBarController:didSelectViewController:',fe
          'tabBarController:shouldSelectViewController:',
          'applicationDidReceiveMemoryWarning:',
          'application:didRegisterForRemoteNotificationsWithDeviceToken:',
          'application:didFailToRegisterForRemoteNotificationsWithError:',
          'application:didReceiveRemoteNotification:fetchCompletionHandler:',
          'application:didRegisterUserNotificationSettings:',
          'application:performActionForShortcutItem:completionHandler:',
          'application:continueUserActivity:restorationHandler:',

          'application:configurationForConnectingSceneSession:options:',
          'application:didDiscardSceneSessions:',
          'application:didFinishLaunchingWithOptions:',
          'scene:willConnectToSession:options:',
          'sceneDidBecomeActive:',
          'sceneDidDisconnect:',
          'sceneDidEnterBackground:',
          'sceneWillEnterForeground:',
          'sceneWillResignActive:',
          'window',
      ].freeze

      # imp -[class sel]

      sub_patten = /[+|-]\[.+\s(.+)\]/
      patten = /\s*imp\s*0x\w*\s*(#{sub_patten})/
      sel_set_patten = /set[A-Z].*:$/
      sel_get_patten = /is[A-Z].*/

      output = `/usr/bin/otool -oV #{path}`

      imp = {}

      output.each_line do |line|
        patten.match(line) do |m|
          sub = sub_patten.match(m[0]) do |subm|

            class_and_sel = subm[0]
            sel = subm[1]

            next if sel.start_with?('.')
            next if apple_protocols.include?(sel)
            next if sel_set_patten.match?(sel)
            next if sel_get_patten.match?(sel)

            if imp.has_key?(sel)
              imp[sel] << class_and_sel
            else
              imp[sel] = [class_and_sel]
            end
          end
        end
      end

      imp.sort
    end

    def reference_selectors(path)
      patten = /__TEXT:__objc_methname:(.+)/
      output = `/usr/bin/otool -v -s __DATA __objc_selrefs #{path}`

      sels = []
      output.each_line do |line|
        patten.match(line) do |m|
          sels << m[1]
        end
      end

      sels
    end
  end
end


module Imp
  class UnusedClass

    include Singleton

    def find_unused_class(path, prefix)
      check_file_type(path)
      result = split_segment_and_find(path, prefix)
      puts Rainbow('below class is unused:').red
      puts result.values
    end

    def check_file_type(path)
      pathname = Pathname.new(path)
      # unless pathname.exist?
      #   raise "#{path} not exit!"
      # end

      cmd = "/usr/bin/file -b #{path}"
      output = `#{cmd}`

      unless output.include?('Mach-O')
        raise 'input file not mach-o file type'
      end
      puts Rainbow('will begin process...').green
      pathname
    end

    def split_segment_and_find(path, prefix)

      arch_command = "lipo -info #{path}"
      arch_output = `#{arch_command}`

      arch = 'arm64'
      if arch_output.include? 'arm64'
        arch = 'arm64'
      elsif arch_output.include? 'x86_64'
        arch = 'x86_64'
      elsif arch_output.include? 'armv7'
        arch = 'armv7'
      end

      command = "/usr/bin/otool -arch #{arch}  -V -o #{path}"
      output = `#{command}`

      class_list_identifier = 'Contents of (__DATA,__objc_classlist) section'
      class_refs_identifier = 'Contents of (__DATA,__objc_classrefs) section'

      unless output.include? class_list_identifier
        raise Rainbow('only support iphone target, please use iphone build...').red
      end

      patten = /Contents of \(.*\) section/

      name_patten_string = '.*'
      unless prefix.empty?
        name_patten_string = "#{prefix}.*"
      end

      vmaddress_to_class_name_patten = /^(\d*\w*)\s(0x\d*\w*)\s_OBJC_CLASS_\$_(#{name_patten_string})/

      class_list = []
      class_refs = []
      used_vmaddress_to_class_name_hash = {}

      can_add_to_list = false
      can_add_to_refs = false

      output.each_line do |line|
        if patten.match?(line)
          if line.include? class_list_identifier
            can_add_to_list = true
            next
          elsif line.include? class_refs_identifier
            can_add_to_list = false
            can_add_to_refs = true
          else
            break
          end
        end

        if can_add_to_list
          class_list << line
        end

        if can_add_to_refs && line
          vmaddress_to_class_name_patten.match(line) do |m|
            unless used_vmaddress_to_class_name_hash[m[2]]
              used_vmaddress_to_class_name_hash[m[2]] = m[3]
            end
          end
        end
      end

      # remove cocoapods class
      podsd_dummy = 'PodsDummy'

      vmaddress_to_class_name_hash = {}
      class_list.each do |line|
        next if line.include? podsd_dummy
        vmaddress_to_class_name_patten.match(line) do |m|
          vmaddress_to_class_name_hash[m[2]] = m[3]
        end
      end

      result = vmaddress_to_class_name_hash
      vmaddress_to_class_name_hash.each do |key, value|
        if used_vmaddress_to_class_name_hash.keys.include?(key)
          result.delete(key)
        end
      end

      result
    end
  end
end



def printUnused(path,pre)
  Imp::UnusedClass.instance.find_unused_class(path,pre)
  Imp::UnusedSel.instance.find_unused_sel(path,pre)
end


# $ARGV = Array['/Users/mega/Desktop/mach-o\ test/MachOTestResult/MachOTest-debug','Un']

path = ARGV[0]
pre = ARGV[1].nil? ? '':ARGV[1]
printUnused(path,pre)
```




