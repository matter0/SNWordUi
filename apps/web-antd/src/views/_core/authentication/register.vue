<script lang="ts" setup>
import type { VbenFormSchema } from '@vben/common-ui';
import type { Recordable } from '@vben/types';
import { computed, h, ref, defineComponent } from 'vue';
import { AuthenticationRegister, z } from '@vben/common-ui';
// 1. 直接引入 Ant Design 的原生组件，确保一定能显示
import { Input, Button } from 'ant-design-vue';
import { message } from 'ant-design-vue'; 
import { sendEmailCode } from '#/api/core/captcha';

defineOptions({ name: 'Register' });

const loading = ref(false);
const timer = ref(0);
const authRegisterRef = ref();

// --- 替换整个 sendCode 函数 ---
async function sendCode() {
  // 1. 获取表单实例，如果获取不到直接返回
  const formApi = authRegisterRef.value?.getFormApi();
  if (!formApi) return;

  try {
    loading.value = true;
    
    // 2. 关键步骤：单独校验 'email' 字段
    // 如果邮箱为空或格式不对，这里会抛错，代码停止向下执行
    await formApi.validateField('email');
    
    // 3. 校验通过后，获取输入的邮箱值
    const values = await formApi.getValues();
    const email = values.email;

    console.log('正在发送验证码，邮箱为:', email);
    // 4. 调用后端 API 发送请求
    await sendEmailCode(email);
    
    // 5. 成功提示
    message.success('验证码已发送');

    // 6. 开始倒计时逻辑 (保持原有逻辑)
    timer.value = 60;
    const interval = setInterval(() => {
      timer.value--;
      if (timer.value <= 0) clearInterval(interval);
    }, 1000);

  } catch (error) {
    // 校验失败会进这里，或者接口报错进这里
    console.error('发送失败或校验未通过', error);
  } finally {
    loading.value = false;
  }
}

// --- 2. 定义一个本地组件：左边输入框，右边按钮 ---
const InputWithCodeBtn = defineComponent({
  name: 'InputWithCodeBtn',
  // 接收表单传递的参数
  props: ['value', 'placeholder'], 
  emits: ['update:value', 'change'],
  setup(props, { emit }) {
    return () => 
      // 使用 div 包裹，flex 布局横向排列
      h('div', { class: 'flex items-center gap-2 w-full' }, [
        // 左侧：Ant Design 原生 Input
        h(Input, {
          value: props.value,
          placeholder: props.placeholder,
          class: 'flex-1', // 占满剩余空间
          // 监听输入，回传给表单
          onInput: (e: any) => emit('update:value', e.target.value),
          onChange: (e: any) => emit('change', e.target.value),
        }),
        // 右侧：发送按钮
        h(
          Button,
          {
            size: 'middle', // 尺寸适中
            disabled: timer.value > 0 || loading.value,
            onClick: sendCode,
            class: 'whitespace-nowrap flex-shrink-0', // 防止被挤压
          },
          () => (timer.value > 0 ? `${timer.value}秒后重试` : '发送验证码'),
        ),
      ]);
  },
});

const formSchema = computed((): VbenFormSchema[] => {
  return [
    // 1. 邮箱
    {
      component: 'VbenInput',
      componentProps: { placeholder: '请输入邮箱地址' },
      fieldName: 'email',
      label: '邮箱',
      rules: z.string().min(1, { message: '请输入邮箱' }).email({ message: '邮箱格式错误' }),
    },
    // 2. 验证码 (使用我们上面定义的组合组件)
    {
      component: InputWithCodeBtn, // <--- 这里直接用组件对象，不要加引号
      componentProps: {
        placeholder: '请输入验证码',
      },
      fieldName: 'code',
      label: '验证码',
      rules: z.string().min(1, { message: '请输入验证码' }),
    },
    // 3. 密码
    {
      component: 'VbenInputPassword',
      componentProps: { placeholder: '设置密码', passwordStrength: true },
      fieldName: 'password',
      label: '密码',
      rules: z.string().min(6, { message: '密码至少6位' }),
    },
    // 4. 确认密码
    {
      component: 'VbenInputPassword',
      componentProps: { placeholder: '确认密码' },
      fieldName: 'confirmPassword',
      label: '确认密码',
      rules: z.string().min(6, { message: '密码至少6位' }).refine((val) => !!val, { message: '两次输入密码不一致' }),
    },
  ];
});

function handleSubmit(value: Recordable<any>) {
  if (value.password !== value.confirmPassword) {
    console.error('两次密码不一致');
    return;
  }
  console.log('register submit:', value);
}
</script>

<template>
  <AuthenticationRegister
    ref="authRegisterRef"
    :form-schema="formSchema"
    :loading="loading"
    @submit="handleSubmit"
  />
</template>
