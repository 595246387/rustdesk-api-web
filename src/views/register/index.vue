<template>
  <div class="login-container">
    <div class="login-card">
      <img src="@/assets/logo.png" alt="logo" class="login-logo"/>
      <div class="register-tip">{{ T('RegisterNeedAdminConfirmTip') }}</div>
      <el-form ref="f" :model="form" label-position="top" class="login-form" :rules="rules">
        <el-form-item :label="T('Username')" prop="username">
          <el-input v-model="form.username" class="login-input"></el-input>
        </el-form-item>

        <el-form-item :label="T('Email')" prop="email">
          <el-input v-model="form.email" class="login-input"></el-input>
        </el-form-item>

        <el-form-item v-if="emailVerify" :label="T('EmailCode')" prop="code">
          <div class="code-row">
            <el-input v-model="form.code" class="login-input" maxlength="6"></el-input>
            <el-button :loading="sending" :disabled="countdown > 0" @click="sendCode">
              {{ countdown > 0 ? T('ResendCodeIn', { seconds: countdown }) : T('SendCode') }}
            </el-button>
          </div>
        </el-form-item>

        <el-form-item :label="T('Password')" prop="password">
          <el-input v-model="form.password" type="password" show-password
                    class="login-input"></el-input>
        </el-form-item>
        <el-form-item :label="T('ConfirmPassword')" prop="confirm_password">
          <el-input v-model="form.confirm_password" type="password" @keyup.enter.native="submit" show-password
                    class="login-input"></el-input>
        </el-form-item>
        <el-form-item label="">
          <el-button @click="submit" class="login-button" type="success">{{ T('Submit') }}</el-button>
          <el-button @click="toLogin" class="login-button">{{ T('ToLogin') }}</el-button>
        </el-form-item>
      </el-form>
    </div>
  </div>
</template>

<script setup>
  import { onMounted, onUnmounted, reactive, ref } from 'vue'
  import { ElMessage } from 'element-plus'
  import { T } from '@/utils/i18n'
  import { useRouter } from 'vue-router'
  import { register, sendRegisterCode } from '@/api/user'
  import { loginOptions } from '@/api/login'
  import { useUserStore } from '@/store/user'
  import { useAppStore } from '@/store/app'

  const router = useRouter()
  const userStore = useUserStore()
  const form = reactive({
    username: '',
    email: '',
    code: '',
    password: '',
    confirm_password: '',
  })
  // emailVerify 由后端 register-email-verify 决定是否显示验证码输入框
  const emailVerify = ref(false)
  const sending = ref(false)
  const countdown = ref(0)
  let countdownTimer = null
  const emailPattern = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  const rules = {
    username: [
      { required: true, message: T('ParamRequired', { param: T('Username') }), trigger: 'blur' },
    ],
    email: [
      {
        validator: (rule, value, callback) => {
          const v = (value || '').trim()
          if (!v) {
            return emailVerify.value
              ? callback(new Error(T('ParamRequired', { param: T('Email') })))
              : callback()
          }
          if (!emailPattern.test(v)) {
            return callback(new Error(T('EmailInvalid')))
          }
          callback()
        }, trigger: 'blur',
      },
    ],
    code: [
      {
        validator: (rule, value, callback) => {
          if (emailVerify.value && !(value || '').trim()) {
            return callback(new Error(T('ParamRequired', { param: T('EmailCode') })))
          }
          callback()
        }, trigger: 'blur',
      },
    ],
    password: [
      { required: true, message: T('ParamRequired', { param: T('Password') }), trigger: 'blur' },
    ],
    confirm_password: [
      { required: true, message: T('ParamRequired', { param: T('ConfirmPassword') }), trigger: 'blur' },
      {
        validator: (rule, value, callback) => {
          if (value !== form.password) {
            callback(new Error(T('PasswordNotMatchConfirmPassword')))
          } else {
            callback()
          }
        }, trigger: 'blur',
      },
    ],
  }
  const f = ref(null)

  onMounted(async () => {
    const res = await loginOptions().catch(_ => false)
    if (res?.data) {
      emailVerify.value = !!res.data.register_email_verify
    }
  })
  onUnmounted(() => {
    if (countdownTimer) {
      clearInterval(countdownTimer)
    }
  })

  const startCountdown = (seconds) => {
    countdown.value = seconds
    if (countdownTimer) {
      clearInterval(countdownTimer)
    }
    countdownTimer = setInterval(() => {
      countdown.value--
      if (countdown.value <= 0) {
        clearInterval(countdownTimer)
        countdownTimer = null
      }
    }, 1000)
  }

  const sendCode = async () => {
    const ok = await f.value.validateField('email').then(() => true).catch(_ => false)
    if (!ok) {
      return
    }
    sending.value = true
    const res = await sendRegisterCode({ email: form.email.trim() }).catch(_ => false)
    sending.value = false
    if (!res) {
      return
    }
    ElMessage.success(T('CodeSent'))
    startCountdown(res.data?.resend_after || 60)
  }

  const submit = async () => {
    const v = await f.value.validate().catch(_ => false)
    if (!v) {
      return
    }
    const payload = { ...form, email: form.email.trim(), code: form.code.trim() }
    const res = await register(payload).catch(_ => false)
    if (!res) {
      return
    }
    if (res.data?.pending) {
      // 需要管理员审核：不自动登录，提示后去登录页。
      ElMessage.success(T('RegisterSuccessWaitAdminConfirm'))
      router.push('/login')
      return
    }
    userStore.saveUserData(res.data)
    useAppStore().loadConfig()
    ElMessage.success(T('RegisterSuccess'))
    router.push('/')
  }
  const toLogin = () => {
    router.push('/login')

  }
</script>

<style scoped lang="scss">
.login-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background-color: #2d3a4b;
  padding: 20px;
  box-sizing: border-box;
}

.login-card {
  width: 360px;
  background-color: #283342;
  padding: 40px;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  text-align: center;
}

h1 {
  margin-bottom: 20px;
  font-size: 24px;
  font-weight: bold;
}

.login-form {
  margin-bottom: 20px;
}

.login-input {
  width: 100%;
}

.login-button {
  width: 100%;
  height: 40px;
  margin-bottom: 20px;
  margin-top: 20px;
  margin-left: 0;
}

.login-logo {
  width: 80px;
  height: 80px;
  margin: 0 auto 20px;
  display: block;
}

.register-tip {
  color: #e6a23c;
  font-size: 12px;
  line-height: 1.5;
  margin-bottom: 12px;
}

.code-row {
  display: flex;
  gap: 8px;
  width: 100%;
  align-items: center;
}

.el-form-item {
  ::v-deep(.el-form-item__label) {
    color: #fff;
  }

  .el-input {
    ::v-deep(.el-input__wrapper) {
      border: 1px solid rgba(255, 255, 255, 0.1);
      background: transparent;
    }

    ::v-deep(input) {
      color: #fff;
    }
  }
}
</style>
