<template>
  <el-form ref="ruleForm2"
           :model="ruleForm2"
           :rules="rules2"
           label-position="left"
           label-width="0px"
           class="demo-ruleForm login-container">
    <h3 class="title">{{ $t('m.Welcome_to_Login') }}</h3>
    <el-form-item prop="account">
      <el-input v-model="ruleForm2.account" :placeholder="$t('m.username')" type="text" auto-complete="off" @keyup.enter.native="handleLogin"></el-input>
    </el-form-item>
    <el-form-item prop="password">
      <el-input v-model="ruleForm2.password" :placeholder="$t('m.password')" type="password" auto-complete="off" @keyup.enter.native="handleLogin"></el-input>
    </el-form-item>
    <el-form-item style="width:100%;">
      <el-button :loading="logining" type="primary" style="width:100%;" @click.native.prevent="handleLogin">{{ $t('m.GO') }}
      </el-button>
    </el-form-item>
  </el-form>
</template>

<script>
  export default {
    data () {
      return {
        logining: false,
        ruleForm2: {
          account: '',
          password: ''
        },
        rules2: {
          account: [
            { required: true, trigger: 'blur' }
          ],
          password: [
            { required: true, trigger: 'blur' }
          ]
        },
        checked: true
      }
    },
    methods: {
      handleLogin (ev) {
        this.$refs.ruleForm2.validate((valid) => {
          if (valid) {
            this.logining = true
            const data = {
              username: this.ruleForm2.account,
              password: this.ruleForm2.password
            }
            this.$store.dispatch('admin/login', data)
              .then(() => {
                this.logining = false
                this.$success('Welcome your login')
                this.$router.push({ name: 'Dashboard' })
              })
              .catch(res => {
                this.$error('You do not have permission to access')
                this.logining = false
              })
          } else {
            this.$error('Please check the error fields')
          }
        })
      }
    }
  }
</script>

<style lang="less" scoped>
  .login-container {
    /*box-shadow: 0 0px 8px 0 rgba(0, 0, 0, 0.06), 0 1px 0px 0 rgba(0, 0, 0, 0.02);*/
    -webkit-border-radius: 5px;
    border-radius: 5px;
    -moz-border-radius: 5px;
    background-clip: padding-box;
    margin: 180px auto;
    width: 350px;
    padding: 35px 35px 15px 35px;
    background: #fff;
    border: 1px solid #eaeaea;
    box-shadow: 0 0 25px #cac6c6;
    .title {
      margin: 0px auto 40px auto;
      text-align: center;
      color: #505458;
    }
    .remember {
      margin: 0px 0px 35px 0px;
    }
  }
</style>
